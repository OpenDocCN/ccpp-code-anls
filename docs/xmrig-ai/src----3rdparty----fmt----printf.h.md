# `xmrig\src\3rdparty\fmt\printf.h`

```
// C++格式化库的printf实现
//
// 版权所有 (c) 2012 - 2016, Victor Zverovich
// 保留所有权利
//
// 有关许可信息，请参阅format.h。

#ifndef FMT_PRINTF_H_
#define FMT_PRINTF_H_

#include <algorithm>  // std::max
#include <limits>     // std::numeric_limits

#include "ostream.h"

FMT_BEGIN_NAMESPACE
namespace detail {

// 检查值是否适合int - 用于避免关于比较有符号和无符号整数的警告
template <bool IsSigned> struct int_checker {
  template <typename T> static bool fits_in_int(T value) {
    unsigned max = max_value<int>();
    return value <= max;
  }
  static bool fits_in_int(bool) { return true; }
};

template <> struct int_checker<true> {
  template <typename T> static bool fits_in_int(T value) {
    return value >= (std::numeric_limits<int>::min)() &&
           value <= max_value<int>();
  }
  static bool fits_in_int(int) { return true; }
};

class printf_precision_handler {
 public:
  // 如果T是整数类型，则返回值，否则抛出异常
  template <typename T, FMT_ENABLE_IF(std::is_integral<T>::value)>
  int operator()(T value) {
    if (!int_checker<std::numeric_limits<T>::is_signed>::fits_in_int(value))
      FMT_THROW(format_error("number is too big"));
    return (std::max)(static_cast<int>(value), 0);
  }

  // 如果T不是整数类型，则抛出异常
  template <typename T, FMT_ENABLE_IF(!std::is_integral<T>::value)>
  int operator()(T) {
    FMT_THROW(format_error("precision is not integer"));
    return 0;
  }
};

// 一个参数访问者，如果arg是零整数则返回true
class is_zero_int {
 public:
  // 如果T是整数类型，则返回值是否为0
  template <typename T, FMT_ENABLE_IF(std::is_integral<T>::value)>
  bool operator()(T value) {
    return value == 0;
  }

  // 如果T不是整数类型，则返回false
  template <typename T, FMT_ENABLE_IF(!std::is_integral<T>::value)>
  bool operator()(T) {
    return false;
  }
};

// 将T转换为无符号整数类型，如果T是bool类型，则返回bool类型
template <typename T> struct make_unsigned_or_bool : std::make_unsigned<T> {};

template <> struct make_unsigned_or_bool<bool> { using type = bool; };
// 定义一个模板类 arg_converter，用于将参数转换为特定类型 T
// T 为目标类型，Context 为上下文类型
template <typename T, typename Context> class arg_converter {
 private:
  // 使用别名定义 char_type 为 Context::char_type 类型
  using char_type = typename Context::char_type;

  // 引用 basic_format_arg<Context> 类型的 arg_，以及 char_type 类型的 type_
  basic_format_arg<Context>& arg_;
  char_type type_;

 public:
  // 构造函数，接受 basic_format_arg<Context> 类型的 arg 和 char_type 类型的 type
  arg_converter(basic_format_arg<Context>& arg, char_type type)
      : arg_(arg), type_(type) {}

  // 重载函数调用操作符，用于将 bool 类型的值转换为目标类型
  void operator()(bool value) {
    // 如果 type_ 不是 's'，则调用 operator()<bool>(value) 进行转换
    if (type_ != 's') operator()<bool>(value);
  }

  // 模板函数，用于将整数类型 U 的值转换为目标类型
  template <typename U, FMT_ENABLE_IF(std::is_integral<U>::value)>
  void operator()(U value) {
    // 判断 type_ 是否为 'd' 或 'i'，确定是否为有符号类型
    bool is_signed = type_ == 'd' || type_ == 'i';
    // 使用 target_type 定义目标类型，如果 T 为 void，则使用 U，否则使用 T
    using target_type = conditional_t<std::is_same<T, void>::value, U, T>;
    // 如果 target_type 的大小不超过 int，则进行转换
    if (const_check(sizeof(target_type) <= sizeof(int))) {
      // 使用额外的转换来消除警告
      if (is_signed) {
        arg_ = detail::make_arg<Context>(
            static_cast<int>(static_cast<target_type>(value)));
      } else {
        using unsigned_type = typename make_unsigned_or_bool<target_type>::type;
        arg_ = detail::make_arg<Context>(
            static_cast<unsigned>(static_cast<unsigned_type>(value)));
      }
    } else {
      if (is_signed) {
        // glibc 的 printf 不会对较小类型的参数进行符号扩展
        // 但我们不必做同样的事情，因为这是未定义行为
        arg_ = detail::make_arg<Context>(static_cast<long long>(value));
      } else {
        arg_ = detail::make_arg<Context>(
            static_cast<typename make_unsigned_or_bool<U>::type>(value));
      }
    }
  }

  // 模板函数，用于非整数类型的值，无需转换
  template <typename U, FMT_ENABLE_IF(!std::is_integral<U>::value)>
  void operator()(U) {}  // 非整数类型无需转换
};

// 将整数参数转换为 T 类型，用于 printf，如果 T 是整数类型
// 如果 T 是 void，则根据类型说明符进行转换：'d' 和 'i' - 有符号，其他 - 无符号
template <typename T, typename Context, typename Char>
void convert_arg(basic_format_arg<Context>& arg, Char type) {
  // 调用 arg_converter 对象的访问函数，将参数转换为指定类型
  visit_format_arg(arg_converter<T, Context>(arg, type), arg);
}

// 用于将整数参数转换为 char 类型以供 printf 使用
template <typename Context> class char_converter {
 private:
  basic_format_arg<Context>& arg_;

 public:
  explicit char_converter(basic_format_arg<Context>& arg) : arg_(arg) {}

  template <typename T, FMT_ENABLE_IF(std::is_integral<T>::value)>
  void operator()(T value) {
    // 将整数值转换为 char 类型，并更新参数
    arg_ = detail::make_arg<Context>(
        static_cast<typename Context::char_type>(value));
  }

  template <typename T, FMT_ENABLE_IF(!std::is_integral<T>::value)>
  void operator()(T) {}  // 针对非整数类型不需要转换
};

// 一个参数访问者，如果参数是字符串则返回指向 C 字符串的指针，否则返回空指针
template <typename Char> struct get_cstring {
  template <typename T> const Char* operator()(T) { return nullptr; }
  const Char* operator()(const Char* s) { return s; }
};

// 检查参数是否是有效的 printf 宽度说明符，并在宽度为负时设置左对齐
template <typename Char> class printf_width_handler {
 private:
  using format_specs = basic_format_specs<Char>;

  format_specs& specs_;

 public:
  explicit printf_width_handler(format_specs& specs) : specs_(specs) {}

  template <typename T, FMT_ENABLE_IF(std::is_integral<T>::value)>
  unsigned operator()(T value) {
    auto width = static_cast<uint32_or_64_or_128_t<T>>(value);
    if (detail::is_negative(value)) {
      specs_.align = align::left;
      width = 0 - width;
    }
    unsigned int_max = max_value<int>();
    if (width > int_max) FMT_THROW(format_error("number is too big"));
    return static_cast<unsigned>(width);
  }

  template <typename T, FMT_ENABLE_IF(!std::is_integral<T>::value)>
  unsigned operator()(T) {
    FMT_THROW(format_error("width is not integer"));
    return 0;
  }
}

template <typename Char, typename Context>
// 定义一个函数 vprintf，用于将格式化后的字符串输出到缓冲区
void vprintf(buffer<Char>& buf, basic_string_view<Char> format,
             basic_format_args<Context> args) {
  // 创建一个上下文对象，用于格式化输出
  Context(buffer_appender<Char>(buf), format, args).format();
}
}  // namespace detail

// 用于向 memory_buffer 输出内容的函数，已废弃
template <typename Char, typename Context>
FMT_DEPRECATED void printf(detail::buffer<Char>& buf,
                           basic_string_view<Char> format,
                           basic_format_args<Context> args) {
  // 调用 vprintf 函数进行格式化输出
  return detail::vprintf(buf, format, args);
}
using detail::vprintf;

// 定义一个基本的 printf 解析上下文类
template <typename Char>
class basic_printf_parse_context : public basic_format_parse_context<Char> {
  using basic_format_parse_context<Char>::basic_format_parse_context;
};
template <typename OutputIt, typename Char> class basic_printf_context;

/**
  \rst
  ``printf`` 参数格式化器。
  \endrst
 */
template <typename OutputIt, typename Char>
class printf_arg_formatter : public detail::arg_formatter_base<OutputIt, Char> {
 public:
  using iterator = OutputIt;

 private:
  using char_type = Char;
  using base = detail::arg_formatter_base<OutputIt, Char>;
  using context_type = basic_printf_context<OutputIt, Char>;

  context_type& context_;

  // 写入空指针的特化版本
  void write_null_pointer(char) {
    this->specs()->type = 0;
    this->write("(nil)");
  }

  void write_null_pointer(wchar_t) {
    this->specs()->type = 0;
    this->write(L"(nil)");
  }

 public:
  using format_specs = typename base::format_specs;

  /**
    \rst
    构造一个参数格式化器对象。
    *buffer* 是输出缓冲区的引用，*specs* 包含标准参数类型的格式说明信息。
    \endrst
   */
  printf_arg_formatter(iterator iter, format_specs& specs, context_type& ctx)
      : base(iter, &specs, detail::locale_ref()), context_(ctx) {}

  // 重载函数调用操作符，用于格式化输出整数类型的参数
  template <typename T, FMT_ENABLE_IF(fmt::detail::is_integral<T>::value)>
  iterator operator()(T value) {
    // MSVC2013 fails to compile separate overloads for bool and char_type so
    // 使用 std::is_same 替代。
    if (std::is_same<T, bool>::value) {
      // 获取格式规范对象的引用
      format_specs& fmt_specs = *this->specs();
      // 如果格式规范对象的类型不是 's'，则返回将 value 转换为 1 或 0 后的结果
      if (fmt_specs.type != 's') return base::operator()(value ? 1 : 0);
      // 将格式规范对象的类型设为 0
      fmt_specs.type = 0;
      // 将 value 写入输出
      this->write(value != 0);
    } else if (std::is_same<T, char_type>::value) {
      // 获取格式规范对象的引用
      format_specs& fmt_specs = *this->specs();
      // 如果格式规范对象的类型存在且不是 'c'，则返回将 value 转换为 int 后的结果
      if (fmt_specs.type && fmt_specs.type != 'c')
        return (*this)(static_cast<int>(value));
      // 将格式规范对象的符号设为无
      fmt_specs.sign = sign::none;
      // 将格式规范对象的备用格式设为假
      fmt_specs.alt = false;
      // 忽略 char 类型的 '0' 标志
      fmt_specs.fill[0] = ' ';
      // 对于非数字类型，需要在此处覆盖 align::numeric，因为对于非数字类型，'0' 标志被忽略
      if (fmt_specs.align == align::none || fmt_specs.align == align::numeric)
        fmt_specs.align = align::right;
      // 返回将 value 写入输出后的结果
      return base::operator()(value);
    } else {
      // 返回将 value 写入输出后的结果
      return base::operator()(value);
    }
    // 返回输出
    return this->out();
  }

  // 如果 T 是浮点类型，则调用此重载函数
  template <typename T, FMT_ENABLE_IF(std::is_floating_point<T>::value)>
  iterator operator()(T value) {
    // 返回将 value 写入输出后的结果
    return base::operator()(value);
  }

  /** 格式化以空字符结尾的 C 字符串。*/
  iterator operator()(const char* value) {
    // 如果 value 存在，则将其写入输出
    if (value)
      base::operator()(value);
    // 如果格式规范对象的类型是 'p'，则写入空指针
    else if (this->specs()->type == 'p')
      write_null_pointer(char_type());
    // 否则写入 "(null)"
    else
      this->write("(null)");
    // 返回输出
    return this->out();
  }

  /** 格式化以空字符结尾的宽字符 C 字符串。*/
  iterator operator()(const wchar_t* value) {
    // 如果 value 存在，则将其写入输出
    if (value)
      base::operator()(value);
    // 如果格式规范对象的类型是 'p'，则写入空指针
    else if (this->specs()->type == 'p')
      write_null_pointer(char_type());
    // 否则写入 L"(null)"
    else
      this->write(L"(null)");
    // 返回输出
    return this->out();
  }

  // 格式化基本字符串视图
  iterator operator()(basic_string_view<char_type> value) {
    // 返回将 value 写入输出后的结果
    return base::operator()(value);
  }

  // 格式化 monostate
  iterator operator()(monostate value) { return base::operator()(value); }

  /** 格式化指针。*/
  iterator operator()(const void* value) {
    // 如果 value 存在，则将其写入输出
    if (value) return base::operator()(value);
    // 将格式规范对象的类型设为 0
    this->specs()->type = 0;
  # 调用 write_null_pointer 函数，传入一个空指针
  write_null_pointer(char_type());
  # 返回当前对象的输出
  return this->out();
}

/** Formats an argument of a custom (user-defined) type. */
iterator operator()(typename basic_format_arg<context_type>::handle handle) {
  # 调用 handle 对象的 format 方法，传入解析上下文和上下文对象
  handle.format(context_.parse_context(), context_);
  # 返回当前对象的输出
  return this->out();
}
};
// 结构体定义结束

template <typename T> struct printf_formatter {
  // 删除默认构造函数

  template <typename ParseContext>
  auto parse(ParseContext& ctx) -> decltype(ctx.begin()) {
    // 解析函数，返回解析上下文的起始位置
    return ctx.begin();
  }

  template <typename FormatContext>
  auto format(const T& value, FormatContext& ctx) -> decltype(ctx.out()) {
    // 格式化函数，将数值通过输出迭代器写入输出
    detail::format_value(detail::get_container(ctx.out()), value);
    return ctx.out();
  }
};

/**
 This template formats data and writes the output through an output iterator.
 */
// 这个模板格式化数据并通过输出迭代器写入输出
template <typename OutputIt, typename Char> class basic_printf_context {
 public:
  /** The character type for the output. */
  using char_type = Char;
  using iterator = OutputIt;
  using format_arg = basic_format_arg<basic_printf_context>;
  using parse_context_type = basic_printf_parse_context<Char>;
  template <typename T> using formatter_type = printf_formatter<T>;

 private:
  using format_specs = basic_format_specs<char_type>;

  OutputIt out_;
  basic_format_args<basic_printf_context> args_;
  parse_context_type parse_ctx_;

  static void parse_flags(format_specs& specs, const Char*& it,
                          const Char* end);

  // Returns the argument with specified index or, if arg_index is -1, the next
  // argument.
  format_arg get_arg(int arg_index = -1);

  // Parses argument index, flags and width and returns the argument index.
  int parse_header(const Char*& it, const Char* end, format_specs& specs);

 public:
  /**
   \rst
   Constructs a ``printf_context`` object. References to the arguments are
   stored in the context object so make sure they have appropriate lifetimes.
   \endrst
   */
  basic_printf_context(OutputIt out, basic_string_view<char_type> format_str,
                       basic_format_args<basic_printf_context> args)
      : out_(out), args_(args), parse_ctx_(format_str) {}

  OutputIt out() { return out_; }
  void advance_to(OutputIt it) { out_ = it; }

  detail::locale_ref locale() { return {}; }

  format_arg arg(int id) const { return args_.get(id); }

  parse_context_type& parse_context() { return parse_ctx_; }

  FMT_CONSTEXPR void on_error(const char* message) {
    parse_ctx_.on_error(message);
  }

  /** Formats stored arguments and writes the output to the range. */
  template <typename ArgFormatter = printf_arg_formatter<OutputIt, Char>>
  OutputIt format();
};

template <typename OutputIt, typename Char>
void basic_printf_context<OutputIt, Char>::parse_flags(format_specs& specs,
                                                       const Char*& it,
                                                       const Char* end) {
  // 遍历格式规范字符串，解析标志位
  for (; it != end; ++it) {
    switch (*it) {
    case '-':
      // 设置左对齐标志
      specs.align = align::left;
      break;
    case '+':
      // 设置正数显示正号标志
      specs.sign = sign::plus;
      break;
    case '0':
      // 设置填充字符为 '0'
      specs.fill[0] = '0';
      break;
    case ' ':
      // 如果未设置正数显示正号标志，则设置为空格标志
      if (specs.sign != sign::plus) {
        specs.sign = sign::space;
      }
      break;
    case '#':
      // 设置备用形式标志
      specs.alt = true;
      break;
    default:
      // 其他情况退出循环
      return;
    }
  }
}

template <typename OutputIt, typename Char>
typename basic_printf_context<OutputIt, Char>::format_arg
basic_printf_context<OutputIt, Char>::get_arg(int arg_index) {
  if (arg_index < 0)
    // 如果参数索引小于0，则获取下一个参数的索引
    arg_index = parse_ctx_.next_arg_id();
  else
    // 否则检查参数索引
    parse_ctx_.check_arg_id(--arg_index);
  // 返回对应参数的格式化参数
  return detail::get_arg(*this, arg_index);
}

template <typename OutputIt, typename Char>
int basic_printf_context<OutputIt, Char>::parse_header(const Char*& it,
                                                       const Char* end,
                                                       format_specs& specs) {
  int arg_index = -1;
  char_type c = *it;
  if (c >= '0' && c <= '9') {
    // 解析参数索引（如果后面跟着'$'）或宽度（可能前面有'0'标志）
    detail::error_handler eh;
    int value = parse_nonnegative_int(it, end, eh);
    if (it != end && *it == '$') {  // value is an argument index
      ++it;
      // 参数索引为解析出的值
      arg_index = value;
    } else {
      if (c == '0') specs.fill[0] = '0';
      if (value != 0) {
        // 非零值表示已解析宽度，不需要再次解析宽度或标志，因此立即返回
        specs.width = value;
        return arg_index;
      }
    }
  }
  // 解析标志位
  parse_flags(specs, it, end);
  // 解析宽度
  if (it != end) {
    # 如果当前字符是数字，则解析非负整数作为字段宽度
    if (*it >= '0' && *it <= '9') {
      # 创建错误处理器对象
      detail::error_handler eh;
      # 解析非负整数作为字段宽度
      specs.width = parse_nonnegative_int(it, end, eh);
    } 
    # 如果当前字符是 '*'，则将字段宽度设置为下一个参数的值
    else if (*it == '*') {
      # 移动到下一个字符
      ++it;
      # 将字段宽度设置为下一个参数的值
      specs.width = static_cast<int>(visit_format_arg(
          detail::printf_width_handler<char_type>(specs), get_arg()));
    }
  }
  # 返回参数索引
  return arg_index;
  // 格式化输出函数模板
template <typename OutputIt, typename Char>
template <typename ArgFormatter>
OutputIt basic_printf_context<OutputIt, Char>::format() {
  // 获取输出迭代器
  auto out = this->out();
  // 获取解析上下文的起始和结束位置
  const Char* start = parse_ctx_.begin();
  const Char* end = parse_ctx_.end();
  auto it = start;
  // 遍历解析上下文
  while (it != end) {
    char_type c = *it++;
    // 如果当前字符不是 '%'，则继续遍历
    if (c != '%') continue;
    // 如果下一个字符也是 '%'，则将 '%' 之前的内容拷贝到输出迭代器中
    if (it != end && *it == c) {
      out = std::copy(start, it, out);
      start = ++it;
      continue;
    }
    // 将 '%' 之前的内容拷贝到输出迭代器中
    out = std::copy(start, it - 1, out);

    // 创建格式化规范对象
    format_specs specs;
    specs.align = align::right;

    // 解析参数索引、标志和宽度
    int arg_index = parse_header(it, end, specs);
    if (arg_index == 0) on_error("argument not found");

    // 解析精度
    if (it != end && *it == '.') {
      ++it;
      c = it != end ? *it : 0;
      if ('0' <= c && c <= '9') {
        detail::error_handler eh;
        specs.precision = parse_nonnegative_int(it, end, eh);
      } else if (c == '*') {
        ++it;
        specs.precision = static_cast<int>(
            visit_format_arg(detail::printf_precision_handler(), get_arg()));
      } else {
        specs.precision = 0;
      }
    }

    // 获取格式化参数
    format_arg arg = get_arg(arg_index);
    // 对于 d、i、o、u、x 和 X 转换说明符，如果指定了精度，则忽略 '0' 标志
    if (specs.precision >= 0 && arg.is_integral())
      specs.fill[0] =
          ' ';  // Ignore '0' flag for non-numeric types or if '-' present.
    // 如果指定了精度，并且参数类型为 cstring_type，则处理字符串
    if (specs.precision >= 0 && arg.type() == detail::type::cstring_type) {
      auto str = visit_format_arg(detail::get_cstring<Char>(), arg);
      auto str_end = str + specs.precision;
      auto nul = std::find(str, str_end, Char());
      arg = detail::make_arg<basic_printf_context>(basic_string_view<Char>(
          str,
          detail::to_unsigned(nul != str_end ? nul - str : specs.precision)));
    }
    // 如果启用了备用格式，则处理零值整数
    if (specs.alt && visit_format_arg(detail::is_zero_int(), arg))
      specs.alt = false;
    // 如果填充字符为'0'
    if (specs.fill[0] == '0') {
      // 如果参数是算术类型且对齐方式不是左对齐，则将对齐方式设置为数字对齐
      if (arg.is_arithmetic() && specs.align != align::left)
        specs.align = align::numeric;
      else
        // 否则，将填充字符设置为' '，忽略'0'标志
        specs.fill[0] = ' ';  // Ignore '0' flag for non-numeric types or if '-' flag is also present.
    }

    // 解析长度并将参数转换为所需的类型
    c = it != end ? *it++ : 0;
    char_type t = it != end ? *it : 0;
    using detail::convert_arg;
    switch (c) {
    case 'h':
      if (t == 'h') {
        ++it;
        t = it != end ? *it : 0;
        convert_arg<signed char>(arg, t);
      } else {
        convert_arg<short>(arg, t);
      }
      break;
    case 'l':
      if (t == 'l') {
        ++it;
        t = it != end ? *it : 0;
        convert_arg<long long>(arg, t);
      } else {
        convert_arg<long>(arg, t);
      }
      break;
    case 'j':
      convert_arg<intmax_t>(arg, t);
      break;
    case 'z':
      convert_arg<size_t>(arg, t);
      break;
    case 't':
      convert_arg<std::ptrdiff_t>(arg, t);
      break;
    case 'L':
      // 当省略'L'时，printf会产生垃圾，因此不需要做相同的事情
      break;
    default:
      --it;
      convert_arg<void>(arg, c);
    }

    // 解析类型
    if (it == end) FMT_THROW(format_error("invalid format string"));
    // 将类型转换为字符并存储在specs.type中
    specs.type = static_cast<char>(*it++);
    if (arg.is_integral()) {
      // 规范化类型
      switch (specs.type) {
      case 'i':
      case 'u':
        specs.type = 'd';
        break;
      case 'c':
        // 如果类型为'c'，则访问格式参数并将参数写入输出
        visit_format_arg(detail::char_converter<basic_printf_context>(arg), arg);
        break;
      }
    }

    // 记录解析的起始位置
    start = it;

    // 格式化参数
    // 访问格式参数并将参数写入输出
    out = visit_format_arg(ArgFormatter(out, specs, *this), arg);
  }
  // 将解析的起始位置到结束位置的内容复制到输出
  return std::copy(start, it, out);
}
// 定义基本的 printf 上下文类型
template <typename Char>
using basic_printf_context_t =
    basic_printf_context<detail::buffer_appender<Char>, Char>;

// 使用 char 类型的基本 printf 上下文类型
using printf_context = basic_printf_context_t<char>;
// 使用 wchar_t 类型的基本 printf 上下文类型
using wprintf_context = basic_printf_context_t<wchar_t>;

// 使用 printf 上下文类型的基本格式参数类型
using printf_args = basic_format_args<printf_context>;
// 使用 wprintf 上下文类型的基本格式参数类型
using wprintf_args = basic_format_args<wprintf_context>;

/**
  \rst
  构造一个 `~fmt::format_arg_store` 对象，其中包含对参数的引用，并可以隐式转换为 `~fmt::printf_args`。
  \endrst
 */
template <typename... Args>
inline format_arg_store<printf_context, Args...> make_printf_args(
    const Args&... args) {
  return {args...};
}

/**
  \rst
  构造一个 `~fmt::format_arg_store` 对象，其中包含对参数的引用，并可以隐式转换为 `~fmt::wprintf_args`。
  \endrst
 */
template <typename... Args>
inline format_arg_store<wprintf_context, Args...> make_wprintf_args(
    const Args&... args) {
  return {args...};
}

// 格式化参数并将结果作为字符串返回
template <typename S, typename Char = char_t<S>>
inline std::basic_string<Char> vsprintf(
    const S& format,
    basic_format_args<basic_printf_context_t<type_identity_t<Char>>> args) {
  basic_memory_buffer<Char> buffer;
  vprintf(buffer, to_string_view(format), args);
  return to_string(buffer);
}

/**
  \rst
  格式化参数并将结果作为字符串返回。

  **示例**::

    std::string message = fmt::sprintf("The answer is %d", 42);
  \endrst
*/
template <typename S, typename... Args,
          typename Char = enable_if_t<detail::is_string<S>::value, char_t<S>>>
inline std::basic_string<Char> sprintf(const S& format, const Args&... args) {
  using context = basic_printf_context_t<Char>;
  return vsprintf(to_string_view(format), make_format_args<context>(args...));
}

template <typename S, typename Char = char_t<S>>
inline int vfprintf(
    std::FILE* f, const S& format,
    # 定义一个函数，接受格式化参数和打印上下文作为输入
    basic_format_args<basic_printf_context_t<type_identity_t<Char>>> args) {
  # 创建一个基本内存缓冲区，用于存储格式化后的内容
  basic_memory_buffer<Char> buffer;
  # 将格式化后的内容写入到内存缓冲区中
  vprintf(buffer, to_string_view(format), args);
  # 获取内存缓冲区的大小
  size_t size = buffer.size();
  # 将内存缓冲区的内容写入到文件流中，并返回写入的字节数
  return std::fwrite(buffer.data(), sizeof(Char), size, f) < size
             ? -1
             : static_cast<int>(size);
}

/**
  \rst
  Prints formatted data to the file *f*.

  **Example**::

    fmt::fprintf(stderr, "Don't %s!", "panic");
  \endrst
 */
template <typename S, typename... Args,
          typename Char = enable_if_t<detail::is_string<S>::value, char_t<S>>>
inline int fprintf(std::FILE* f, const S& format, const Args&... args) {
  using context = basic_printf_context_t<Char>;
  return vfprintf(f, to_string_view(format),
                  make_format_args<context>(args...));
}

template <typename S, typename Char = char_t<S>>
inline int vprintf(
    const S& format,
    basic_format_args<basic_printf_context_t<type_identity_t<Char>>> args) {
  return vfprintf(stdout, to_string_view(format), args);
}

/**
  \rst
  Prints formatted data to ``stdout``.

  **Example**::

    fmt::printf("Elapsed time: %.2f seconds", 1.23);
  \endrst
 */
template <typename S, typename... Args,
          FMT_ENABLE_IF(detail::is_string<S>::value)>
inline int printf(const S& format_str, const Args&... args) {
  using context = basic_printf_context_t<char_t<S>>;
  return vprintf(to_string_view(format_str),
                 make_format_args<context>(args...));
}

template <typename S, typename Char = char_t<S>>
inline int vfprintf(
    std::basic_ostream<Char>& os, const S& format,
    basic_format_args<basic_printf_context_t<type_identity_t<Char>>> args) {
  basic_memory_buffer<Char> buffer;
  vprintf(buffer, to_string_view(format), args);
  detail::write_buffer(os, buffer);
  return static_cast<int>(buffer.size());
}

/** Formats arguments and writes the output to the range. */
template <typename ArgFormatter, typename Char,
          typename Context =
              basic_printf_context<typename ArgFormatter::iterator, Char>>
typename ArgFormatter::iterator vprintf(
    detail::buffer<Char>& out, basic_string_view<Char> format_str,
  # 使用模板参数 type_identity_t<Context> 实例化 basic_format_args
  basic_format_args<type_identity_t<Context>> args) {
  # 创建 ArgFormatter 类型的迭代器 iter，指向输出流 out
  typename ArgFormatter::iterator iter(out);
  # 使用 format_str 和 args 实例化 Context 对象，调用 ArgFormatter 的 format 方法进行格式化
  Context(iter, format_str, args).template format<ArgFormatter>();
  # 返回迭代器
  return iter;
// 结束 FMT 命名空间
}

/**
  \rst
  将格式化数据打印到流 *os* 中。

  **示例**::

    fmt::fprintf(cerr, "Don't %s!", "panic");
  \endrst
 */
// 使用模板定义 fprintf 函数，接受流 os、格式化字符串 format_str 和参数 args
template <typename S, typename... Args, typename Char = char_t<S>>
// 内联函数，返回值为整型
inline int fprintf(std::basic_ostream<Char>& os, const S& format_str,
                   const Args&... args) {
  // 使用 context 定义基本的 printf 上下文
  using context = basic_printf_context_t<Char>;
  // 调用 vfprintf 函数，将格式化字符串转换为视图，使用 make_format_args 函数创建格式化参数
  return vfprintf(os, to_string_view(format_str),
                  make_format_args<context>(args...));
}
// 结束 FMT 命名空间
FMT_END_NAMESPACE

// 结束条件编译，关闭 FMT_PRINTF_H_ 宏定义
#endif  // FMT_PRINTF_H_
```