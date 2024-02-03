# `xmrig\src\3rdparty\fmt\compile.h`

```cpp
// 格式化库的 C++ 实验性格式字符串编译
//
// 版权所有，Victor Zverovich 和 fmt 贡献者，2012 年至今
// 保留所有权利
//
// 有关许可信息，请参阅 format.h。

#ifndef FMT_COMPILE_H_
#define FMT_COMPILE_H_

#include <vector>

#include "format.h"

FMT_BEGIN_NAMESPACE
namespace detail {

// 编译成快速格式化代码的编译时字符串
class compiled_string {};

template <typename S>
struct is_compiled_string : std::is_base_of<compiled_string, S> {};

/**
  \rst
  将字符串字面量 *s* 转换为格式字符串，该字符串将在编译时解析并转换为高效的格式化代码。需要 C++17 ``constexpr if`` 编译器支持。

  **示例**::

    // 使用最有效的方法将 42 转换为 std::string，无需运行时格式字符串处理。
    std::string s = fmt::format(FMT_COMPILE("{}"), 42);
  \endrst
 */
#define FMT_COMPILE(s) FMT_STRING_IMPL(s, fmt::detail::compiled_string)

template <typename T, typename... Tail>
const T& first(const T& value, const Tail&...) {
  return value;
}

// 编译格式字符串的一部分。它可以是文字文本，也可以是替换字段。
template <typename Char> struct format_part {
  enum class kind { arg_index, arg_name, text, replacement };

  struct replacement {
    arg_ref<Char> arg_id;
    dynamic_format_specs<Char> specs;
  };

  kind part_kind;
  union value {
    int arg_index;
    basic_string_view<Char> str;
    replacement repl;

    FMT_CONSTEXPR value(int index = 0) : arg_index(index) {}
    FMT_CONSTEXPR value(basic_string_view<Char> s) : str(s) {}
    FMT_CONSTEXPR value(replacement r) : repl(r) {}
  } val;
  // 参数 id 结束位置
  const Char* arg_id_end = nullptr;

  FMT_CONSTEXPR format_part(kind k = kind::arg_index, value v = {})
      : part_kind(k), val(v) {}

  static FMT_CONSTEXPR format_part make_arg_index(int index) {
    # 返回一个格式化部分，根据参数索引和索引值
    return format_part(kind::arg_index, index);
  }
  # 创建一个参数名格式化部分，根据给定的名称
  static FMT_CONSTEXPR format_part make_arg_name(basic_string_view<Char> name) {
    return format_part(kind::arg_name, name);
  }
  # 创建一个文本格式化部分，根据给定的文本内容
  static FMT_CONSTEXPR format_part make_text(basic_string_view<Char> text) {
    return format_part(kind::text, text);
  }
  # 创建一个替换格式化部分，根据给定的替换值
  static FMT_CONSTEXPR format_part make_replacement(replacement repl) {
    return format_part(kind::replacement, repl);
  }
};

// 定义一个模板结构体，用于计算格式字符串中的部分数量
template <typename Char> struct part_counter {
  unsigned num_parts = 0;

  // 处理文本部分
  FMT_CONSTEXPR void on_text(const Char* begin, const Char* end) {
    if (begin != end) ++num_parts;
  }

  // 处理参数 ID
  FMT_CONSTEXPR int on_arg_id() { return ++num_parts, 0; }
  FMT_CONSTEXPR int on_arg_id(int) { return ++num_parts, 0; }
  FMT_CONSTEXPR int on_arg_id(basic_string_view<Char>) {
    return ++num_parts, 0;
  }

  // 处理替换字段
  FMT_CONSTEXPR void on_replacement_field(int, const Char*) {}

  // 处理格式规范
  FMT_CONSTEXPR const Char* on_format_specs(int, const Char* begin,
                                            const Char* end) {
    // 查找匹配的大括号
    unsigned brace_counter = 0;
    for (; begin != end; ++begin) {
      if (*begin == '{') {
        ++brace_counter;
      } else if (*begin == '}') {
        if (brace_counter == 0u) break;
        --brace_counter;
      }
    }
    return begin;
  }

  // 处理错误
  FMT_CONSTEXPR void on_error(const char*) {}
};

// 计算格式字符串中的部分数量
template <typename Char>
FMT_CONSTEXPR unsigned count_parts(basic_string_view<Char> format_str) {
  part_counter<Char> counter;
  parse_format_string<true>(format_str, counter);
  return counter.num_parts;
}

// 格式字符串编译器类
template <typename Char, typename PartHandler>
class format_string_compiler : public error_handler {
 private:
  using part = format_part<Char>;

  PartHandler handler_;
  part part_;
  basic_string_view<Char> format_str_;
  basic_format_parse_context<Char> parse_context_;

 public:
  // 构造函数
  FMT_CONSTEXPR format_string_compiler(basic_string_view<Char> format_str,
                                       PartHandler handler)
      : handler_(handler),
        format_str_(format_str),
        parse_context_(format_str) {}

  // 处理文本部分
  FMT_CONSTEXPR void on_text(const Char* begin, const Char* end) {
    if (begin != end)
      handler_(part::make_text({begin, to_unsigned(end - begin)}));
  }

  // 处理参数 ID
  FMT_CONSTEXPR int on_arg_id() {
    part_ = part::make_arg_index(parse_context_.next_arg_id());
  }
  // 返回整数 0
  return 0;
}

// 处理位置参数的整数 id
FMT_CONSTEXPR int on_arg_id(int id) {
  // 检查位置参数 id 的有效性
  parse_context_.check_arg_id(id);
  // 创建 part 对象，表示位置参数的索引
  part_ = part::make_arg_index(id);
  // 返回整数 0
  return 0;
}

// 处理位置参数的字符串 id
FMT_CONSTEXPR int on_arg_id(basic_string_view<Char> id) {
  // 创建 part 对象，表示位置参数的名称
  part_ = part::make_arg_name(id);
  // 返回整数 0
  return 0;
}

// 处理替换字段
FMT_CONSTEXPR void on_replacement_field(int, const Char* ptr) {
  // 设置 part_ 的 arg_id_end 属性
  part_.arg_id_end = ptr;
  // 调用 handler_ 处理 part_ 对象
  handler_(part_);
}

// 处理格式规范
FMT_CONSTEXPR const Char* on_format_specs(int, const Char* begin,
                                          const Char* end) {
  // 创建 replacement 对象
  auto repl = typename part::replacement();
  // 创建动态规范处理器
  dynamic_specs_handler<basic_format_parse_context<Char>> handler(
      repl.specs, parse_context_);
  // 解析格式规范
  auto it = parse_format_specs(begin, end, handler);
  // 如果解析结果不是 '}'，则报错
  if (*it != '}') on_error("missing '}' in format string");
  // 根据 part_ 的类型，设置 replacement 对象的 arg_id
  repl.arg_id = part_.part_kind == part::kind::arg_index
                    ? arg_ref<Char>(part_.val.arg_index)
                    : arg_ref<Char>(part_.val.str);
  // 创建替换字段的 part 对象
  auto part = part::make_replacement(repl);
  // 设置 part 的 arg_id_end 属性
  part.arg_id_end = begin;
  // 调用 handler_ 处理 part 对象
  handler_(part);
  // 返回解析结果的迭代器
  return it;
}
};

// 编译格式字符串并为每个解析的部分调用处理程序（part）
template <bool IS_CONSTEXPR, typename Char, typename PartHandler>
FMT_CONSTEXPR void compile_format_string(basic_string_view<Char> format_str,
                                         PartHandler handler) {
  // 解析格式字符串
  parse_format_string<IS_CONSTEXPR>(
      format_str,
      format_string_compiler<Char, PartHandler>(format_str, handler));
}

// 格式化参数
template <typename OutputIt, typename Context, typename Id>
void format_arg(
    basic_format_parse_context<typename Context::char_type>& parse_ctx,
    Context& ctx, Id arg_id) {
  // 前进到访问格式参数
  ctx.advance_to(visit_format_arg(
      arg_formatter<OutputIt, typename Context::char_type>(ctx, &parse_ctx),
      ctx.arg(arg_id)));
}

// vformat_to定义在子命名空间中以防止ADL（Argument-Dependent Lookup）
namespace cf {
template <typename Context, typename OutputIt, typename CompiledFormat>
auto vformat_to(OutputIt out, CompiledFormat& cf,
                basic_format_args<Context> args) -> typename Context::iterator {
  using char_type = typename Context::char_type;
  basic_format_parse_context<char_type> parse_ctx(
      to_string_view(cf.format_str_));
  Context ctx(out, args);

  const auto& parts = cf.parts();
  for (auto part_it = std::begin(parts); part_it != std::end(parts);
       ++part_it) {
    const auto& part = *part_it;
    const auto& value = part.val;

    using format_part_t = format_part<char_type>;
    switch (part.part_kind) {
    case format_part_t::kind::text: {
      // 处理文本部分
      const auto text = value.str;
      auto output = ctx.out();
      auto&& it = reserve(output, text.size());
      it = std::copy_n(text.begin(), text.size(), it);
      ctx.advance_to(output);
      break;
    }

    case format_part_t::kind::arg_index:
      // 前进到解析上下文的参数索引结束位置
      advance_to(parse_ctx, part.arg_id_end);
      // 格式化参数
      detail::format_arg<OutputIt>(parse_ctx, ctx, value.arg_index);
      break;
    // 如果格式部分的类型是 arg_name
    case format_part_t::kind::arg_name:
      // 将解析上下文移动到参数结束位置
      advance_to(parse_ctx, part.arg_id_end);
      // 调用 detail::format_arg 函数处理参数值并输出到输出迭代器
      detail::format_arg<OutputIt>(parse_ctx, ctx, value.str);
      // 跳出 switch 语句
      break;

    // 如果格式部分的类型是 replacement
    case format_part_t::kind::replacement: {
      // 获取参数值的引用
      const auto& arg_id_value = value.repl.arg_id.val;
      // 根据参数值的类型获取参数
      const auto arg = value.repl.arg_id.kind == arg_id_kind::index
                           ? ctx.arg(arg_id_value.index)
                           : ctx.arg(arg_id_value.name);

      // 获取替换部分的规格
      auto specs = value.repl.specs;

      // 处理宽度规格
      handle_dynamic_spec<width_checker>(specs.width, specs.width_ref, ctx);
      // 处理精度规格
      handle_dynamic_spec<precision_checker>(specs.precision,
                                             specs.precision_ref, ctx);

      // 创建错误处理器
      error_handler h;
      // 创建数字规格检查器
      numeric_specs_checker<error_handler> checker(h, arg.type());
      // 如果对齐方式是数字型，则要求参数是数字类型
      if (specs.align == align::numeric) checker.require_numeric_argument();
      // 如果有符号位，则检查参数的符号
      if (specs.sign != sign::none) checker.check_sign();
      // 如果使用备用格式，则要求参数是数字类型
      if (specs.alt) checker.require_numeric_argument();
      // 如果有精度规格，则检查参数的精度
      if (specs.precision >= 0) checker.check_precision();

      // 将解析上下文移动到参数结束位置
      advance_to(parse_ctx, part.arg_id_end);
      // 将格式化参数的输出迭代器移动到指定位置
      ctx.advance_to(
          visit_format_arg(arg_formatter<OutputIt, typename Context::char_type>(
                               ctx, nullptr, &specs),
                           arg));
      // 跳出 switch 语句
      break;
    }
    }
  }
  // 返回输出迭代器
  return ctx.out();
}  // namespace cf
}  // namespace cf

// 定义基本编译格式结构体
struct basic_compiled_format {};

// 定义编译格式基类模板
template <typename S, typename = void>
struct compiled_format_base : basic_compiled_format {
  using char_type = char_t<S>;
  using parts_container = std::vector<detail::format_part<char_type>>;

  parts_container compiled_parts;

  // 根据格式字符串编译成格式部分
  explicit compiled_format_base(basic_string_view<char_type> format_str) {
    compile_format_string<false>(format_str,
                                 [this](const format_part<char_type>& part) {
                                   compiled_parts.push_back(part);
                                 });
  }

  // 返回编译后的格式部分
  const parts_container& parts() const { return compiled_parts; }
};

// 定义格式部分数组模板
template <typename Char, unsigned N> struct format_part_array {
  format_part<Char> data[N] = {};
  FMT_CONSTEXPR format_part_array() = default;
};

// 将格式字符串编译成格式部分数组
template <typename Char, unsigned N>
FMT_CONSTEXPR format_part_array<Char, N> compile_to_parts(
    basic_string_view<Char> format_str) {
  format_part_array<Char, N> parts;
  unsigned counter = 0;
  // This is not a lambda for compatibility with older compilers.
  struct {
    format_part<Char>* parts;
    unsigned* counter;
    FMT_CONSTEXPR void operator()(const format_part<Char>& part) {
      parts[(*counter)++] = part;
    }
  } collector{parts.data, &counter};
  compile_format_string<true>(format_str, collector);
  if (counter < N) {
    parts.data[counter] =
        format_part<Char>::make_text(basic_string_view<Char>());
  }
  return parts;
}

// 返回两个值中的最大值
template <typename T> constexpr const T& constexpr_max(const T& a, const T& b) {
  return (a < b) ? b : a;
}

// 定义编译格式基类模板（针对编译时字符串）
template <typename S>
struct compiled_format_base<S, enable_if_t<is_compile_string<S>::value>>
    : basic_compiled_format {
  using char_type = char_t<S>;

  FMT_CONSTEXPR explicit compiled_format_base(basic_string_view<char_type>) {}

// Workaround for old compilers. Format string compilation will not be
// performed there anyway.
#if FMT_USE_CONSTEXPR
  // 如果支持 constexpr，则使用 constexpr_max 计算格式部分的数量
  static FMT_CONSTEXPR_DECL const unsigned num_format_parts =
      constexpr_max(count_parts(to_string_view(S())), 1u);
#else
  // 否则，格式部分数量为 1
  static const unsigned num_format_parts = 1;
#endif

  // 使用格式部分数量创建格式部分容器
  using parts_container = format_part<char_type>[num_format_parts];

  // 返回格式部分容器的引用
  const parts_container& parts() const {
    // 编译格式字符串为格式部分，并存储为静态变量
    static FMT_CONSTEXPR_DECL const auto compiled_parts =
        compile_to_parts<char_type, num_format_parts>(
            detail::to_string_view(S()));
    // 返回编译后的格式部分数据
    return compiled_parts.data;
  }
};

template <typename S, typename... Args>
class compiled_format : private compiled_format_base<S> {
 public:
  using typename compiled_format_base<S>::char_type;

 private:
  basic_string_view<char_type> format_str_;

  template <typename Context, typename OutputIt, typename CompiledFormat>
  friend auto cf::vformat_to(OutputIt out, CompiledFormat& cf,
                             basic_format_args<Context> args) ->
      typename Context::iterator;

 public:
  // 禁用默认构造函数
  compiled_format() = delete;
  // 显式构造函数，接受格式字符串
  explicit constexpr compiled_format(basic_string_view<char_type> format_str)
      : compiled_format_base<S>(format_str), format_str_(format_str) {}
};

#ifdef __cpp_if_constexpr
template <typename... Args> struct type_list {};

// 返回 [first, rest...] 中索引为 N 的参数的引用
template <int N, typename T, typename... Args>
constexpr const auto& get([[maybe_unused]] const T& first,
                          [[maybe_unused]] const Args&... rest) {
  static_assert(N < 1 + sizeof...(Args), "index is out of bounds");
  if constexpr (N == 0)
    return first;
  else
    return get<N - 1>(rest...);
}

// 获取索引为 N 的参数的类型
template <int N, typename> struct get_type_impl;

template <int N, typename... Args> struct get_type_impl<N, type_list<Args...>> {
  using type = remove_cvref_t<decltype(get<N>(std::declval<Args>()...))>;
};

template <int N, typename T>
using get_type = typename get_type_impl<N, T>::type;

// 检查类型是否为编译格式类型
template <typename T> struct is_compiled_format : std::false_type {};
// 定义一个模板结构体 text，包含一个基于模板参数的字符串视图和一个字符类型别名
template <typename Char> struct text {
  basic_string_view<Char> data;
  using char_type = Char;

  // 定义一个格式化方法，将数据写入输出迭代器
  template <typename OutputIt, typename... Args>
  OutputIt format(OutputIt out, const Args&...) const {
    return write<Char>(out, data);
  }
};

// 指明 text<Char> 是一个已编译格式
template <typename Char>
struct is_compiled_format<text<Char>> : std::true_type {};

// 创建一个 text<Char> 对象，包含从字符串视图中提取的数据
template <typename Char>
constexpr text<Char> make_text(basic_string_view<Char> s, size_t pos,
                               size_t size) {
  return {{&s[pos], size}};
}

// 定义一个模板结构体 code_unit，包含一个基于模板参数的值和一个字符类型别名
template <typename Char> struct code_unit {
  Char value;
  using char_type = Char;

  // 定义一个格式化方法，将值写入输出迭代器
  template <typename OutputIt, typename... Args>
  OutputIt format(OutputIt out, const Args&...) const {
    return write<Char>(out, value);
  }
};

// 指明 code_unit<Char> 是一个已编译格式
template <typename Char>
struct is_compiled_format<code_unit<Char>> : std::true_type {};

// 定义一个模板结构体 field，表示一个引用第 N 个参数的替换字段
template <typename Char, typename T, int N> struct field {
  using char_type = Char;

  // 定义一个格式化方法，将参数值写入输出迭代器
  template <typename OutputIt, typename... Args>
  OutputIt format(OutputIt out, const Args&... args) const {
    // 确保参数类型可以转换为 const T&
    const T& arg = get<N>(args...);
    return write<Char>(out, arg);
  }
};

// 指明 field<Char, T, N> 是一个已编译格式
template <typename Char, typename T, int N>
struct is_compiled_format<field<Char, T, N>> : std::true_type {};

// 定义一个模板结构体 spec_field，表示一个带有格式说明符的替换字段，引用第 N 个参数
template <typename Char, typename T, int N> struct spec_field {
  using char_type = Char;
  mutable formatter<T, Char> fmt;

  // 定义一个格式化方法，使用格式说明符格式化参数值并写入输出迭代器
  template <typename OutputIt, typename... Args>
  OutputIt format(OutputIt out, const Args&... args) const {
    // 确保参数类型可以转换为 const T&
    const T& arg = get<N>(args...);
    const auto& vargs =
        make_format_args<basic_format_context<OutputIt, Char>>(args...);
    basic_format_context<OutputIt, Char> ctx(out, vargs);
    return fmt.format(arg, ctx);
  }
};

// 指明 spec_field<Char, T, N> 是一个已编译格式
template <typename Char, typename T, int N>
struct is_compiled_format<spec_field<Char, T, N>> : std::true_type {};
// 检查 spec_field<Char, T, N> 是否为编译格式，如果是则返回 true_type
template <typename L, typename R>
struct is_compiled_format<spec_field<Char, T, N>> : std::true_type {};

// 定义 concat 结构体，包含左右两个成员和字符类型
template <typename L, typename R>
struct concat {
  L lhs;
  R rhs;
  using char_type = typename L::char_type;

  // 格式化函数，将左右两个成员格式化输出到指定位置
  template <typename OutputIt, typename... Args>
  OutputIt format(OutputIt out, const Args&... args) const {
    out = lhs.format(out, args...);
    return rhs.format(out, args...);
  }
};

// 检查 concat<L, R> 是否为编译格式，如果是则返回 true_type
template <typename L, typename R>
struct is_compiled_format<concat<L, R>> : std::true_type {};

// 创建 concat 对象的函数，接受左右两个成员并返回一个 concat 对象
template <typename L, typename R>
constexpr concat<L, R> make_concat(L lhs, R rhs) {
  return {lhs, rhs};
}

// 定义 unknown_format 结构体
struct unknown_format {};

// 解析文本内容的函数，返回解析后的位置
template <typename Char>
constexpr size_t parse_text(basic_string_view<Char> str, size_t pos) {
  for (size_t size = str.size(); pos != size; ++pos) {
    if (str[pos] == '{' || str[pos] == '}') break;
  }
  return pos;
}

// 编译格式字符串的函数模板，接受参数 Args、位置 POS、ID 和字符串 S
template <typename Args, size_t POS, int ID, typename S>
constexpr auto compile_format_string(S format_str);

// 解析尾部内容的函数模板，接受头部内容 head 和字符串 format_str
template <typename Args, size_t POS, int ID, typename T, typename S>
constexpr auto parse_tail(T head, S format_str) {
  if constexpr (POS !=
                basic_string_view<typename S::char_type>(format_str).size()) {
    constexpr auto tail = compile_format_string<Args, POS, ID>(format_str);
    if constexpr (std::is_same<remove_cvref_t<decltype(tail)>,
                               unknown_format>())
      return tail;
    else
      return make_concat(head, tail);
  } else {
    return head;
  }
}

// 解析格式规范的结果结构体，包含格式化器、结束位置和下一个参数 ID
template <typename T, typename Char>
struct parse_specs_result {
  formatter<T, Char> fmt;
  size_t end;
  int next_arg_id;
};

// 解析格式规范的函数模板，接受字符串 str、位置 pos 和参数 ID
template <typename T, typename Char>
constexpr parse_specs_result<T, Char> parse_specs(basic_string_view<Char> str,
                                                  size_t pos, int arg_id) {
  str.remove_prefix(pos);
  auto ctx = basic_format_parse_context<Char>(str, {}, arg_id + 1);
  auto f = formatter<T, Char>();
  auto end = f.parse(ctx);
  return {f, pos + (end - str.data()) + 1, ctx.next_arg_id()};
}
// 编译非空格式字符串并返回编译后的表示，如果输入无法识别则返回 unknown_format()
template <typename Args, size_t POS, int ID, typename S>
constexpr auto compile_format_string(S format_str) {
  using char_type = typename S::char_type;
  // 将格式字符串转换为字符视图
  constexpr basic_string_view<char_type> str = format_str;
  if constexpr (str[POS] == '{') {
    if (POS + 1 == str.size())
      throw format_error("unmatched '{' in format string");
    if constexpr (str[POS + 1] == '{') {
      return parse_tail<Args, POS + 2, ID>(make_text(str, POS, 1), format_str);
    } else if constexpr (str[POS + 1] == '}') {
      using type = get_type<ID, Args>;
      return parse_tail<Args, POS + 2, ID + 1>(field<char_type, type, ID>(),
                                               format_str);
    } else if constexpr (str[POS + 1] == ':') {
      using type = get_type<ID, Args>;
      // 解析格式规范
      constexpr auto result = parse_specs<type>(str, POS + 2, ID);
      return parse_tail<Args, result.end, result.next_arg_id>(
          spec_field<char_type, type, ID>{result.fmt}, format_str);
    } else {
      return unknown_format();
    }
  } else if constexpr (str[POS] == '}') {
    if (POS + 1 == str.size())
      throw format_error("unmatched '}' in format string");
    return parse_tail<Args, POS + 2, ID>(make_text(str, POS, 1), format_str);
  } else {
    // 解析文本内容
    constexpr auto end = parse_text(str, POS + 1);
    if constexpr (end - POS > 1) {
      return parse_tail<Args, end, ID>(make_text(str, POS, end - POS),
                                       format_str);
    } else {
      return parse_tail<Args, end, ID>(code_unit<char_type>{str[POS]},
                                       format_str);
    }
  }
}

template <typename... Args, typename S,
          FMT_ENABLE_IF(is_compile_string<S>::value ||
                        detail::is_compiled_string<S>::value)>
// 使用编译时字符串格式化，返回编译后的格式化对象
constexpr auto compile(S format_str) {
  // 将传入的格式化字符串转换为基本字符串视图
  constexpr basic_string_view<typename S::char_type> str = format_str;
  // 如果字符串长度为0，则返回一个空文本对象
  if constexpr (str.size() == 0) {
    return detail::make_text(str, 0, 0);
  } else {
    // 否则，编译格式化字符串并返回结果
    constexpr auto result =
        detail::compile_format_string<detail::type_list<Args...>, 0, 0>(
            format_str);
    // 如果编译结果为未知格式，则返回编译后的格式化对象
    if constexpr (std::is_same<remove_cvref_t<decltype(result)>,
                               detail::unknown_format>()) {
      return detail::compiled_format<S, Args...>(to_string_view(format_str));
    } else {
      // 否则返回编译结果
      return result;
    }
  }
}
#else
// 如果不支持编译时字符串格式化，则使用运行时字符串格式化
template <typename... Args, typename S,
          FMT_ENABLE_IF(is_compile_string<S>::value)>
constexpr auto compile(S format_str) -> detail::compiled_format<S, Args...> {
  return detail::compiled_format<S, Args...>(to_string_view(format_str));
}
#endif  // __cpp_if_constexpr

// 编译字符串格式化，格式化字符串必须是字符串字面量
template <typename... Args, typename Char, size_t N>
auto compile(const Char (&format_str)[N])
    -> detail::compiled_format<const Char*, Args...> {
  return detail::compiled_format<const Char*, Args...>(
      basic_string_view<Char>(format_str, N - 1));
}
}  // namespace detail

// 已弃用！使用 FMT_COMPILE 代替
template <typename... Args>
FMT_DEPRECATED auto compile(const Args&... args)
    -> decltype(detail::compile(args...)) {
  return detail::compile(args...);
}

#if FMT_USE_CONSTEXPR
#  ifdef __cpp_if_constexpr

// 格式化编译后的格式化对象和参数
template <typename CompiledFormat, typename... Args,
          typename Char = typename CompiledFormat::char_type,
          FMT_ENABLE_IF(detail::is_compiled_format<CompiledFormat>::value)>
FMT_INLINE std::basic_string<Char> format(const CompiledFormat& cf,
                                          const Args&... args) {
  // 创建基本内存缓冲区
  basic_memory_buffer<Char> buffer;
  // 使用格式化对象对参数进行格式化
  cf.format(detail::buffer_appender<Char>(buffer), args...);
  // 将格式化后的结果转换为字符串并返回
  return to_string(buffer);
}
template <typename OutputIt, typename CompiledFormat, typename... Args,
          FMT_ENABLE_IF(detail::is_compiled_format<CompiledFormat>::value)>
OutputIt format_to(OutputIt out, const CompiledFormat& cf,
                   const Args&... args) {
  return cf.format(out, args...);
}
#  endif  // __cpp_if_constexpr
#endif    // FMT_USE_CONSTEXPR

template <typename CompiledFormat, typename... Args,
          typename Char = typename CompiledFormat::char_type,
          FMT_ENABLE_IF(std::is_base_of<detail::basic_compiled_format,
                                        CompiledFormat>::value)>
std::basic_string<Char> format(const CompiledFormat& cf, const Args&... args) {
  basic_memory_buffer<Char> buffer;
  using context = buffer_context<Char>;
  detail::cf::vformat_to<context>(detail::buffer_appender<Char>(buffer), cf,
                                  make_format_args<context>(args...));
  return to_string(buffer);
}

template <typename S, typename... Args,
          FMT_ENABLE_IF(detail::is_compiled_string<S>::value)>
FMT_INLINE std::basic_string<typename S::char_type> format(const S&,
                                                           Args&&... args) {
#ifdef __cpp_if_constexpr
  if constexpr (std::is_same<typename S::char_type, char>::value) {
    constexpr basic_string_view<typename S::char_type> str = S();
    if (str.size() == 2 && str[0] == '{' && str[1] == '}')
      return fmt::to_string(detail::first(args...));
  }
#endif
  constexpr auto compiled = detail::compile<Args...>(S());
  return format(compiled, std::forward<Args>(args)...);
}

template <typename OutputIt, typename CompiledFormat, typename... Args,
          FMT_ENABLE_IF(std::is_base_of<detail::basic_compiled_format,
                                        CompiledFormat>::value)>
// 格式化输出到输出迭代器，使用预编译格式和参数
template <typename OutputIt, typename CompiledFormat, typename... Args>
OutputIt format_to(OutputIt out, const CompiledFormat& cf, const Args&... args) {
  // 定义字符类型
  using char_type = typename CompiledFormat::char_type;
  // 定义格式化上下文
  using context = format_context_t<OutputIt, char_type>;
  // 调用内部函数进行格式化输出
  return detail::cf::vformat_to<context>(out, cf, make_format_args<context>(args...));
}

// 如果参数是预编译字符串类型，则调用此重载函数
template <typename OutputIt, typename S, typename... Args,
          FMT_ENABLE_IF(detail::is_compiled_string<S>::value)>
OutputIt format_to(OutputIt out, const S&, const Args&... args) {
  // 编译字符串参数
  constexpr auto compiled = detail::compile<Args...>(S());
  // 调用上面的格式化函数
  return format_to(out, compiled, args...);
}

// 格式化输出到输出迭代器，限制输出字符数量
template <
    typename OutputIt, typename CompiledFormat, typename... Args,
    FMT_ENABLE_IF(detail::is_output_iterator<OutputIt>::value&& std::is_base_of<
                  detail::basic_compiled_format, CompiledFormat>::value)>
format_to_n_result<OutputIt> format_to_n(OutputIt out, size_t n,
                                         const CompiledFormat& cf,
                                         const Args&... args) {
  // 使用截断迭代器限制输出字符数量
  auto it = format_to(detail::truncating_iterator<OutputIt>(out, n), cf, args...);
  return {it.base(), it.count()};
}

// 如果参数是预编译字符串类型，则调用此重载函数，限制输出字符数量
template <typename OutputIt, typename S, typename... Args,
          FMT_ENABLE_IF(detail::is_compiled_string<S>::value)>
format_to_n_result<OutputIt> format_to_n(OutputIt out, size_t n, const S&,
                                         const Args&... args) {
  // 编译字符串参数
  constexpr auto compiled = detail::compile<Args...>(S());
  // 使用截断迭代器限制输出字符数量
  auto it = format_to(detail::truncating_iterator<OutputIt>(out, n), compiled, args...);
  return {it.base(), it.count()};
}

// 获取格式化输出的字符数量
template <typename CompiledFormat, typename... Args>
size_t formatted_size(const CompiledFormat& cf, const Args&... args) {
  // 调用格式化函数并返回字符数量
  return format_to(detail::counting_iterator(), cf, args...).count();
}

// 结束命名空间
FMT_END_NAMESPACE

// 结束条件编译
#endif  // FMT_COMPILE_H_
```