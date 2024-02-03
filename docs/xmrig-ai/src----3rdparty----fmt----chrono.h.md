# `xmrig\src\3rdparty\fmt\chrono.h`

```cpp
// 格式化库的 C++ - 时间支持
//
// 版权所有 (c) 2012 - 现在，Victor Zverovich
// 保留所有权利
//
// 有关许可信息，请参阅 format.h。

#ifndef FMT_CHRONO_H_
#define FMT_CHRONO_H_

#include <chrono>  // 包含时间库
#include <ctime>   // 包含时间库
#include <locale>  // 包含本地化库
#include <sstream> // 包含字符串流库

#include "format.h"  // 包含格式化库
#include "locale.h"  // 包含本地化库

FMT_BEGIN_NAMESPACE  // 开始 FMT 命名空间

// 启用安全的时间持续，除非显式禁用。
#ifndef FMT_SAFE_DURATION_CAST
#  define FMT_SAFE_DURATION_CAST 1
#endif
#if FMT_SAFE_DURATION_CAST

// 用于在 std::chrono::durations 之间进行转换，避免未定义行为或错误结果。
// 这是 fmt 中的 duration_cast 的简化版本。
// 有关详细信息，请参阅 https://github.com/pauldreik/safe_duration_cast
//
// 版权所有 Paul Dreik 2019
namespace safe_duration_cast {

template <typename To, typename From,
          FMT_ENABLE_IF(!std::is_same<From, To>::value &&
                        std::numeric_limits<From>::is_signed ==
                            std::numeric_limits<To>::is_signed)>
FMT_CONSTEXPR To lossless_integral_conversion(const From from, int& ec) {
  ec = 0;
  using F = std::numeric_limits<From>;
  using T = std::numeric_limits<To>;
  static_assert(F::is_integer, "From must be integral");
  static_assert(T::is_integer, "To must be integral");

  // A and B are both signed, or both unsigned.
  if (F::digits <= T::digits) {
    // From fits in To without any problem.
  } else {
    // From does not always fit in To, resort to a dynamic check.
    if (from < (T::min)() || from > (T::max)()) {
      // outside range.
      ec = 1;
      return {};
    }
  }
  return static_cast<To>(from);
}

/**
 * 将 From 转换为 To，无损失。如果动态值 from 无法无损转换为 To，则设置 ec。
 */
template <typename To, typename From,
          FMT_ENABLE_IF(!std::is_same<From, To>::value &&
                        std::numeric_limits<From>::is_signed !=
                            std::numeric_limits<To>::is_signed)>
FMT_CONSTEXPR To lossless_integral_conversion(const From from, int& ec) {
  ec = 0;
  using F = std::numeric_limits<From>;
  using T = std::numeric_limits<To>;
  static_assert(F::is_integer, "From must be integral");
  static_assert(T::is_integer, "To must be integral");

  if (detail::const_check(F::is_signed && !T::is_signed)) {
    // From may be negative, not allowed!
    if (fmt::detail::is_negative(from)) {
      ec = 1;
      return {};
    }
    // From is positive. Can it always fit in To?
    if (F::digits > T::digits &&
        from > static_cast<From>(detail::max_value<To>())) {
      ec = 1;
      return {};
    }
  }

  if (!F::is_signed && T::is_signed && F::digits >= T::digits &&
      from > static_cast<From>(detail::max_value<To>())) {
    ec = 1;
    return {};
  }
  return static_cast<To>(from);  // Lossless conversion.
}

template <typename To, typename From,
          FMT_ENABLE_IF(std::is_same<From, To>::value)>
FMT_CONSTEXPR To lossless_integral_conversion(const From from, int& ec) {
  ec = 0;
  return from;
}  // function

// clang-format off
/**
 * converts From to To if possible, otherwise ec is set.
 *
 * input                            |    output
 * ---------------------------------|---------------
 * NaN                              | NaN
 * Inf                              | Inf
 * normal, fits in output           | converted (possibly lossy)
 * normal, does not fit in output   | ec is set
 * subnormal                        | best effort
 * -Inf                             | -Inf
 */
// clang-format on
template <typename To, typename From,
          FMT_ENABLE_IF(!std::is_same<From, To>::value)>
FMT_CONSTEXPR To safe_float_conversion(const From from, int& ec) {
  ec = 0;
  using T = std::numeric_limits<To>;
  static_assert(std::is_floating_point<From>::value, "From must be floating");
  static_assert(std::is_floating_point<To>::value, "To must be floating");

  // catch the only happy case
  // 捕获唯一的正常情况
  if (std::isfinite(from)) {
    if (from >= T::lowest() && from <= (T::max)()) {
      return static_cast<To>(from);
    }
    // not within range.
    // 不在范围内
    ec = 1;
    return {};
  }

  // nan and inf will be preserved
  // nan 和 inf 将被保留
  return static_cast<To>(from);
}  // function

template <typename To, typename From,
          FMT_ENABLE_IF(std::is_same<From, To>::value)>
FMT_CONSTEXPR To safe_float_conversion(const From from, int& ec) {
  ec = 0;
  static_assert(std::is_floating_point<From>::value, "From must be floating");
  return from;
}

/**
 * safe duration cast between integral durations
 */
// 在整数持续时间之间进行安全转换
template <typename To, typename FromRep, typename FromPeriod,
          FMT_ENABLE_IF(std::is_integral<FromRep>::value),
          FMT_ENABLE_IF(std::is_integral<typename To::rep>::value)>
// 安全地将时长从一个类型转换为另一个类型
template <typename To, typename FromRep, typename FromPeriod,
          FMT_ENABLE_IF(std::is_floating_point<FromRep>::value),
          FMT_ENABLE_IF(std::is_floating_point<typename To::rep>::value)>
To safe_duration_cast(std::chrono::duration<FromRep, FromPeriod> from,
                      int& ec) {
  using From = std::chrono::duration<FromRep, FromPeriod>;
  // 初始化错误码为0
  ec = 0;
  // 基本思想是，我们需要通过将来自类型的count()转换为目标类型的count()，通过乘以这个因子：
  struct Factor
      : std::ratio_divide<typename From::period, typename To::period> {};

  // 确保Factor::num是正数
  static_assert(Factor::num > 0, "num must be positive");
  // 确保Factor::den是正数
  static_assert(Factor::den > 0, "den must be positive");

  // 转换的过程是这样的：将from.count()乘以Factor::num/Factor::den，并将其转换为To::rep类型，所有这些都不会发生溢出/下溢。让我们首先找到一个适合的类型，可以容纳To、From和Factor::num
  using IntermediateRep =
      typename std::common_type<typename From::rep, typename To::rep,
                                decltype(Factor::num)>::type;

  // 安全地转换为IntermediateRep类型
  IntermediateRep count =
      lossless_integral_conversion<IntermediateRep>(from.count(), ec);
  // 如果有错误，返回空值
  if (ec) return {};
  // 如果Factor::num不等于1，则乘以Factor::num而不会发生溢出或下溢
  if (detail::const_check(Factor::num != 1)) {
    const auto max1 = detail::max_value<IntermediateRep>() / Factor::num;
    if (count > max1) {
      ec = 1;
      return {};
    }
    const auto min1 =
        (std::numeric_limits<IntermediateRep>::min)() / Factor::num;
    if (count < min1) {
      ec = 1;
      return {};
    }
    count *= Factor::num;
  }

  // 如果Factor::den不等于1，则除以Factor::den
  if (detail::const_check(Factor::den != 1)) count /= Factor::den;
  // 将count安全地转换为To::rep类型
  auto tocount = lossless_integral_conversion<typename To::rep>(count, ec);
  // 如果有错误，返回空值；否则返回转换后的时长
  return ec ? To() : To(tocount);
}
# 定义一个函数，将一个时间段从一种类型转换为另一种类型，并返回转换后的结果
template <class To, class FromRep, class FromPeriod>
To safe_duration_cast(std::chrono::duration<FromRep, FromPeriod> from,
                      int& ec) {
  using From = std::chrono::duration<FromRep, FromPeriod>;
  # 初始化错误码为0
  ec = 0;
  # 如果输入的时间段是 NaN，则直接返回 To 类型的 quiet_NaN
  if (std::isnan(from.count())) {
    // nan in, gives nan out. easy.
    return To{std::numeric_limits<typename To::rep>::quiet_NaN()};
  }
  # 如果输入的时间段是正无穷或负无穷，则直接返回对应的 To 类型
  // +-inf should be preserved.
  if (std::isinf(from.count())) {
    return To{from.count()};
  }

  # 计算转换因子，用于将 From 类型的 count 转换为 To 类型的 count
  struct Factor
      : std::ratio_divide<typename From::period, typename To::period> {};

  # 确保转换因子的分子和分母都是正数
  static_assert(Factor::num > 0, "num must be positive");
  static_assert(Factor::den > 0, "den must be positive");

  # 找到一个适合存储 To、From 和 Factor::num 的中间类型
  using IntermediateRep =
      typename std::common_type<typename From::rep, typename To::rep,
                                decltype(Factor::num)>::type;

  # 强制将 From::rep 转换为 IntermediateRep 类型，即使在这种情况下它永远不会发生缩小转换
  IntermediateRep count =
      safe_float_conversion<IntermediateRep>(from.count(), ec);
  if (ec) {
    return {};
  }

  # 使用转换因子 Factor::num 进行乘法，确保没有溢出或下溢
  if (Factor::num != 1) {
    constexpr auto max1 = detail::max_value<IntermediateRep>() /
                          static_cast<IntermediateRep>(Factor::num);
    if (count > max1) {
      ec = 1;
      return {};
    }
    constexpr auto min1 = std::numeric_limits<IntermediateRep>::lowest() /
                          static_cast<IntermediateRep>(Factor::num);
    if (count < min1) {
      ec = 1;
      return {};
    }
  // 将 count 乘以 Factor::num，并将结果赋给 count
  count *= static_cast<IntermediateRep>(Factor::num);
}

// 这不会出错，对吧？den>0 在之前已经检查过了。
if (Factor::den != 1) {
  // 定义一个通用类型 common_t，用于存储 IntermediateRep 和 intmax_t 的公共类型
  using common_t = typename std::common_type<IntermediateRep, intmax_t>::type;
  // 将 count 除以 Factor::den，并将结果赋给 count
  count /= static_cast<common_t>(Factor::den);
}

// 将 count 转换为 To 类型的 rep 类型，安全地进行转换
using ToRep = typename To::rep;

// 将 count 安全地转换为 ToRep 类型的变量 tocount，并检查是否出错
const ToRep tocount = safe_float_conversion<ToRep>(count, ec);
if (ec) {
  // 如果出错，返回空值
  return {};
}
// 返回转换后的 To 类型对象
return To{tocount};
}  // 结束 safe_duration_cast 命名空间

}  // 结束 if 预处理指令

// 防止前面的标记作为函数风格宏进行扩展
// 用法：f FMT_NOMACRO()
#define FMT_NOMACRO

// detail 命名空间
namespace detail {
// localtime_r 函数的宏定义
inline null<> localtime_r FMT_NOMACRO(...) { return null<>(); }
// localtime_s 函数的宏定义
inline null<> localtime_s(...) { return null<>(); }
// gmtime_r 函数的宏定义
inline null<> gmtime_r(...) { return null<>(); }
// gmtime_s 函数的宏定义
inline null<> gmtime_s(...) { return null<>(); }
}  // 结束 detail 命名空间

// 线程安全的 std::localtime 替代品
inline std::tm localtime(std::time_t time) {
  // dispatcher 结构体
  struct dispatcher {
    std::time_t time_;
    std::tm tm_;

    dispatcher(std::time_t t) : time_(t) {}

    // 运行函数
    bool run() {
      using namespace fmt::detail;
      return handle(localtime_r(&time_, &tm_));
    }

    // 处理函数
    bool handle(std::tm* tm) { return tm != nullptr; }

    // 处理函数
    bool handle(detail::null<>) {
      using namespace fmt::detail;
      return fallback(localtime_s(&tm_, &time_));
    }

    // 回退函数
    bool fallback(int res) { return res == 0; }

    // 回退函数
#if !FMT_MSC_VER
    bool fallback(detail::null<>) {
      using namespace fmt::detail;
      std::tm* tm = std::localtime(&time_);
      if (tm) tm_ = *tm;
      return tm != nullptr;
    }
#endif
  };
  dispatcher lt(time);
  // 时间值过大可能不受支持
  if (!lt.run()) FMT_THROW(format_error("time_t value out of range"));
  return lt.tm_;
}

// 线程安全的 std::gmtime 替代品
inline std::tm gmtime(std::time_t time) {
  // dispatcher 结构体
  struct dispatcher {
    std::time_t time_;
    std::tm tm_;

    dispatcher(std::time_t t) : time_(t) {}

    // 运行函数
    bool run() {
      using namespace fmt::detail;
      return handle(gmtime_r(&time_, &tm_));
    }

    // 处理函数
    bool handle(std::tm* tm) { return tm != nullptr; }

    // 处理函数
    bool handle(detail::null<>) {
      using namespace fmt::detail;
      return fallback(gmtime_s(&tm_, &time_));
    }
    # 定义一个名为fallback的函数，接受一个整数参数res，返回一个布尔值
    bool fallback(int res) { return res == 0; }
#if !FMT_MSC_VER
    // 如果不是 MSC_VER 编译器，则使用 null 作为参数调用 fallback 函数
    bool fallback(detail::null<>) {
      // 将 time_ 转换为 UTC 时间，并存储到 tm 结构体中
      std::tm* tm = std::gmtime(&time_);
      // 如果 tm 不为空，则将 tm_ 更新为 tm
      if (tm) tm_ = *tm;
      // 返回 tm 是否为空的结果
      return tm != nullptr;
    }
#endif
  };
  // 创建 dispatcher 对象 gt，传入 time 作为参数
  dispatcher gt(time);
  // 如果时间值过大，则不支持，抛出 format_error 异常
  if (!gt.run()) FMT_THROW(format_error("time_t value out of range"));
  // 返回 gt 的 tm_ 成员变量
  return gt.tm_;
}

// 根据时间点返回对应的 UTC 时间
inline std::tm gmtime(
    std::chrono::time_point<std::chrono::system_clock> time_point) {
  return gmtime(std::chrono::system_clock::to_time_t(time_point));
}

namespace detail {
// 根据格式化字符串和时间结构体，返回格式化后的字符串长度
inline size_t strftime(char* str, size_t count, const char* format,
                       const std::tm* time) {
  return std::strftime(str, count, format, time);
}

// 根据格式化字符串和时间结构体，返回格式化后的宽字符字符串长度
inline size_t strftime(wchar_t* str, size_t count, const wchar_t* format,
                       const std::tm* time) {
  return std::wcsftime(str, count, format, time);
}
}  // namespace detail

// 时间点的格式化输出
template <typename Char>
struct formatter<std::chrono::time_point<std::chrono::system_clock>, Char>
    : formatter<std::tm, Char> {
  template <typename FormatContext>
  auto format(std::chrono::time_point<std::chrono::system_clock> val,
              FormatContext& ctx) -> decltype(ctx.out()) {
    // 将时间点转换为本地时间
    std::tm time = localtime(val);
    // 调用 formatter<std::tm, Char> 的 format 函数进行格式化输出
    return formatter<std::tm, Char>::format(time, ctx);
  }
};

// 时间结构体的格式化输出
template <typename Char> struct formatter<std::tm, Char> {
  // 解析格式化字符串
  template <typename ParseContext>
  auto parse(ParseContext& ctx) -> decltype(ctx.begin()) {
    auto it = ctx.begin();
    // 如果解析到 ':'，则向后移动一个位置
    if (it != ctx.end() && *it == ':') ++it;
    auto end = it;
    // 查找格式化字符串的结束位置
    while (end != ctx.end() && *end != '}') ++end;
    // 将格式化字符串存储到 tm_format 中
    tm_format.reserve(detail::to_unsigned(end - it + 1));
    tm_format.append(it, end);
    tm_format.push_back('\0');
    return end;
  }

  // 格式化输出时间结构体
  template <typename FormatContext>
  auto format(const std::tm& tm, FormatContext& ctx) -> decltype(ctx.out()) {
    basic_memory_buffer<Char> buf;
    size_t start = buf.size();
    // 无限循环，直到满足条件跳出循环
    for (;;) {
      // 计算剩余缓冲区大小
      size_t size = buf.capacity() - start;
      // 使用 strftime 函数将时间格式化为字符串，存储到 buf 中
      size_t count = detail::strftime(&buf[start], size, &tm_format[0], &tm);
      // 如果成功格式化时间，则调整 buf 大小，跳出循环
      if (count != 0) {
        buf.resize(start + count);
        break;
      }
      // 如果缓冲区大小超过格式化字符串大小的 256 倍，则假定 strftime 返回空结果，跳出循环
      if (size >= tm_format.size() * 256) {
        // 如果缓冲区大小超过格式化字符串大小的 256 倍，则假定 strftime 返回空结果
        // 没有更好的方法来区分这两种情况：
        // https://github.com/fmtlib/fmt/issues/367
        break;
      }
      // 定义最小增长值
      const size_t MIN_GROWTH = 10;
      // 如果需要，增加缓冲区大小
      buf.reserve(buf.capacity() + (size > MIN_GROWTH ? size : MIN_GROWTH));
    }
    // 将 buf 中的内容复制到输出流中，并返回
    return std::copy(buf.begin(), buf.end(), ctx.out());
  }

  // 定义存储时间格式的缓冲区
  basic_memory_buffer<Char> tm_format;
};
// 命名空间 detail

// 返回时间单位的字符串表示
template <typename Period> FMT_CONSTEXPR const char* get_units() {
  return nullptr;
}
// 返回时间单位的字符串表示，特化为 atto 秒
template <> FMT_CONSTEXPR const char* get_units<std::atto>() { return "as"; }
// 返回时间单位的字符串表示，特化为 femto 秒
template <> FMT_CONSTEXPR const char* get_units<std::femto>() { return "fs"; }
// 返回时间单位的字符串表示，特化为 pico 秒
template <> FMT_CONSTEXPR const char* get_units<std::pico>() { return "ps"; }
// 返回时间单位的字符串表示，特化为 nano 秒
template <> FMT_CONSTEXPR const char* get_units<std::nano>() { return "ns"; }
// 返回时间单位的字符串表示，特化为 micro 秒
template <> FMT_CONSTEXPR const char* get_units<std::micro>() { return "µs"; }
// 返回时间单位的字符串表示，特化为 milli 秒
template <> FMT_CONSTEXPR const char* get_units<std::milli>() { return "ms"; }
// 返回时间单位的字符串表示，特化为 centi 秒
template <> FMT_CONSTEXPR const char* get_units<std::centi>() { return "cs"; }
// 返回时间单位的字符串表示，特化为 deci 秒
template <> FMT_CONSTEXPR const char* get_units<std::deci>() { return "ds"; }
// 返回时间单位的字符串表示，特化为 ratio<1> 秒
template <> FMT_CONSTEXPR const char* get_units<std::ratio<1>>() { return "s"; }
// 返回时间单位的字符串表示，特化为 deca 秒
template <> FMT_CONSTEXPR const char* get_units<std::deca>() { return "das"; }
// 返回时间单位的字符串表示，特化为 hecto 秒
template <> FMT_CONSTEXPR const char* get_units<std::hecto>() { return "hs"; }
// 返回时间单位的字符串表示，特化为 kilo 秒
template <> FMT_CONSTEXPR const char* get_units<std::kilo>() { return "ks"; }
// 返回时间单位的字符串表示，特化为 mega 秒
template <> FMT_CONSTEXPR const char* get_units<std::mega>() { return "Ms"; }
// 返回时间单位的字符串表示，特化为 giga 秒
template <> FMT_CONSTEXPR const char* get_units<std::giga>() { return "Gs"; }
// 返回时间单位的字符串表示，特化为 tera 秒
template <> FMT_CONSTEXPR const char* get_units<std::tera>() { return "Ts"; }
// 返回时间单位的字符串表示，特化为 peta 秒
template <> FMT_CONSTEXPR const char* get_units<std::peta>() { return "Ps"; }
// 返回时间单位的字符串表示，特化为 exa 秒
template <> FMT_CONSTEXPR const char* get_units<std::exa>() { return "Es"; }
// 返回时间单位的字符串表示，特化为 ratio<60> 秒
template <> FMT_CONSTEXPR const char* get_units<std::ratio<60>>() {
  return "m";
}
// 返回时间单位的字符串表示，特化为 ratio<3600> 秒
template <> FMT_CONSTEXPR const char* get_units<std::ratio<3600>>() {
  return "h";
}

// 数字系统枚举类
enum class numeric_system {
  standard,
  // 替代数字系统，例如在 ja_JP 区域设置中使用 "十二" 代替 "12"
  alternative
};

// 解析类似 put_time 的格式字符串，并调用处理程序动作
template <typename Char, typename Handler>
FMT_CONSTEXPR const Char* parse_chrono_format(const Char* begin,
                                              const Char* end,
                                              Handler&& handler) {
  // 从开始位置开始遍历格式字符串，直到结束位置
  auto ptr = begin;
  while (ptr != end) {
    // 获取当前字符
    auto c = *ptr;
    // 如果当前字符为 '}'，则跳出循环
    if (c == '}') break;
    // 如果当前字符不是 '%'，则继续下一个字符
    if (c != '%') {
      ++ptr;
      continue;
    }
    // 如果当前位置不是开始位置，则将文本部分传递给处理器
    if (begin != ptr) handler.on_text(begin, ptr);
    // 消耗 '%'，移动到下一个字符
    ++ptr;  // consume '%'
    // 如果已经到达结束位置，则抛出格式错误异常
    if (ptr == end) FMT_THROW(format_error("invalid format"));
    // 获取下一个字符
    c = *ptr++;
    // 根据不同的格式字符，调用处理器的不同方法
    switch (c) {
    case '%':
      handler.on_text(ptr - 1, ptr);
      break;
    case 'n': {
      const Char newline[] = {'\n'};
      handler.on_text(newline, newline + 1);
      break;
    }
    case 't': {
      const Char tab[] = {'\t'};
      handler.on_text(tab, tab + 1);
      break;
    }
    // Day of the week:
    case 'a':
      handler.on_abbr_weekday();
      break;
    case 'A':
      handler.on_full_weekday();
      break;
    case 'w':
      handler.on_dec0_weekday(numeric_system::standard);
      break;
    case 'u':
      handler.on_dec1_weekday(numeric_system::standard);
      break;
    // Month:
    case 'b':
      handler.on_abbr_month();
      break;
    case 'B':
      handler.on_full_month();
      break;
    // Hour, minute, second:
    case 'H':
      handler.on_24_hour(numeric_system::standard);
      break;
    case 'I':
      handler.on_12_hour(numeric_system::standard);
      break;
    case 'M':
      handler.on_minute(numeric_system::standard);
      break;
    case 'S':
      handler.on_second(numeric_system::standard);
      break;
    // Other:
    case 'c':
      handler.on_datetime(numeric_system::standard);
      break;
    case 'x':
      handler.on_loc_date(numeric_system::standard);
      break;
    case 'X':
      handler.on_loc_time(numeric_system::standard);
      break;
    case 'D':
      handler.on_us_date();
      break;
    case 'F':
      handler.on_iso_date();
      break;
    # 根据不同的格式字符执行不同的处理函数
    case 'r':
      # 处理12小时制时间
      handler.on_12_hour_time();
      break;
    case 'R':
      # 处理24小时制时间
      handler.on_24_hour_time();
      break;
    case 'T':
      # 处理ISO时间
      handler.on_iso_time();
      break;
    case 'p':
      # 处理上午下午标识
      handler.on_am_pm();
      break;
    case 'Q':
      # 处理持续时间数值
      handler.on_duration_value();
      break;
    case 'q':
      # 处理持续时间单位
      handler.on_duration_unit();
      break;
    case 'z':
      # 处理UTC偏移量
      handler.on_utc_offset();
      break;
    case 'Z':
      # 处理时区名称
      handler.on_tz_name();
      break;
    # 备用表示：
    case 'E': {
      # 如果指针已经到达结尾，抛出格式错误异常
      if (ptr == end) FMT_THROW(format_error("invalid format"));
      # 读取下一个字符
      c = *ptr++;
      # 根据不同的字符执行不同的处理函数
      switch (c) {
      case 'c':
        # 处理日期时间（备用表示）
        handler.on_datetime(numeric_system::alternative);
        break;
      case 'x':
        # 处理本地日期（备用表示）
        handler.on_loc_date(numeric_system::alternative);
        break;
      case 'X':
        # 处理本地时间（备用表示）
        handler.on_loc_time(numeric_system::alternative);
        break;
      default:
        # 抛出格式错误异常
        FMT_THROW(format_error("invalid format"));
      }
      break;
    }
    case 'O':
      # 如果指针已经到达结尾，抛出格式错误异常
      if (ptr == end) FMT_THROW(format_error("invalid format"));
      # 读取下一个字符
      c = *ptr++;
      # 根据不同的字符执行不同的处理函数
      switch (c) {
      case 'w':
        # 处理星期几（备用表示）
        handler.on_dec0_weekday(numeric_system::alternative);
        break;
      case 'u':
        # 处理星期几（备用表示）
        handler.on_dec1_weekday(numeric_system::alternative);
        break;
      case 'H':
        # 处理24小时制时间（备用表示）
        handler.on_24_hour(numeric_system::alternative);
        break;
      case 'I':
        # 处理12小时制时间（备用表示）
        handler.on_12_hour(numeric_system::alternative);
        break;
      case 'M':
        # 处理分钟（备用表示）
        handler.on_minute(numeric_system::alternative);
        break;
      case 'S':
        # 处理秒（备用表示）
        handler.on_second(numeric_system::alternative);
        break;
      default:
        # 抛出格式错误异常
        FMT_THROW(format_error("invalid format"));
      }
      break;
    default:
      # 抛出格式错误异常
      FMT_THROW(format_error("invalid format"));
    }
    # 更新指针位置
    begin = ptr;
  }
  # 如果指针位置有变化，处理文本内容
  if (begin != ptr) handler.on_text(begin, ptr);
  # 返回指针位置
  return ptr;
// 结构体 chrono_format_checker，用于检查日期格式
struct chrono_format_checker {
  // 报告没有日期的错误
  FMT_NORETURN void report_no_date() { FMT_THROW(format_error("no date")); }

  // 处理文本类型的日期
  template <typename Char> void on_text(const Char*, const Char*) {}
  // 处理缩写星期几
  FMT_NORETURN void on_abbr_weekday() { report_no_date(); }
  // 处理完整星期几
  FMT_NORETURN void on_full_weekday() { report_no_date(); }
  // 处理十进制星期几
  FMT_NORETURN void on_dec0_weekday(numeric_system) { report_no_date(); }
  // 处理十进制星期几
  FMT_NORETURN void on_dec1_weekday(numeric_system) { report_no_date(); }
  // 处理缩写月份
  FMT_NORETURN void on_abbr_month() { report_no_date(); }
  // 处理完整月份
  FMT_NORETURN void on_full_month() { report_no_date(); }
  // 处理24小时制时间
  void on_24_hour(numeric_system) {}
  // 处理12小时制时间
  void on_12_hour(numeric_system) {}
  // 处理分钟
  void on_minute(numeric_system) {}
  // 处理秒
  void on_second(numeric_system) {}
  // 处理日期时间
  FMT_NORETURN void on_datetime(numeric_system) { report_no_date(); }
  // 处理本地日期
  FMT_NORETURN void on_loc_date(numeric_system) { report_no_date(); }
  // 处理本地时间
  FMT_NORETURN void on_loc_time(numeric_system) { report_no_date(); }
  // 处理美国日期
  FMT_NORETURN void on_us_date() { report_no_date(); }
  // 处理ISO日期
  FMT_NORETURN void on_iso_date() { report_no_date(); }
  // 处理12小时制时间
  void on_12_hour_time() {}
  // 处理24小时制时间
  void on_24_hour_time() {}
  // 处理ISO时间
  void on_iso_time() {}
  // 处理上午/下午
  void on_am_pm() {}
  // 处理持续时间数值
  void on_duration_value() {}
  // 处理持续时间单位
  void on_duration_unit() {}
  // 处理UTC偏移
  FMT_NORETURN void on_utc_offset() { report_no_date(); }
  // 处理时区名称
  FMT_NORETURN void on_tz_name() { report_no_date(); }
};

// 将值转换为整数，并检查其是否在范围[0, upper)内
template <typename T, FMT_ENABLE_IF(std::is_integral<T>::value)>
inline bool isnan(T) {
  return false;
}
// 将值转换为浮点数，并检查其是否为NaN
template <typename T, FMT_ENABLE_IF(std::is_floating_point<T>::value)>
inline bool isnan(T value) {
  return std::isnan(value);
}

// 将值转换为整数，并检查其是否为有限数
template <typename T, FMT_ENABLE_IF(std::is_integral<T>::value)>
inline bool isfinite(T) {
  return true;
}
// 将值转换为浮点数，并检查其是否为有限数
template <typename T, FMT_ENABLE_IF(std::is_floating_point<T>::value)>
inline bool isfinite(T value) {
  return std::isfinite(value);
}
// 将值转换为非负整数，如果值不是整数类型，则使用浮点数版本
inline int to_nonnegative_int(T value, int upper) {
  // 断言值大于等于0且小于等于上限，否则输出错误信息
  FMT_ASSERT(value >= 0 && value <= upper, "invalid value");
  // 忽略上限参数
  (void)upper;
  // 将值转换为整数类型并返回
  return static_cast<int>(value);
}
// 如果值不是整数类型，则使用浮点数版本
template <typename T, FMT_ENABLE_IF(!std::is_integral<T>::value)>
inline int to_nonnegative_int(T value, int upper) {
  // 断言值为NaN或者大于等于0且小于等于上限，否则输出错误信息
  FMT_ASSERT(
      std::isnan(value) || (value >= 0 && value <= static_cast<T>(upper)),
      "invalid value");
  // 忽略上限参数
  (void)upper;
  // 将值转换为整数类型并返回
  return static_cast<int>(value);
}

// 如果T是整数类型，则返回T的无符号类型，否则返回T本身（与std::make_unsigned不同）
template <typename T, bool INTEGRAL = std::is_integral<T>::value>
struct make_unsigned_or_unchanged {
  using type = T;
};
// 如果T是整数类型，则返回T的无符号类型
template <typename T> struct make_unsigned_or_unchanged<T, true> {
  using type = typename std::make_unsigned<T>::type;
};

#if FMT_SAFE_DURATION_CAST
// 抛出异常的安全时间转换版本
template <typename To, typename FromRep, typename FromPeriod>
To fmt_safe_duration_cast(std::chrono::duration<FromRep, FromPeriod> from) {
  // 错误码
  int ec;
  // 调用safe_duration_cast::safe_duration_cast进行安全时间转换
  To to = safe_duration_cast::safe_duration_cast<To>(from, ec);
  // 如果有错误，抛出格式化错误异常
  if (ec) FMT_THROW(format_error("cannot format duration"));
  // 返回转换后的值
  return to;
}
#endif

// 获取时间间隔的毫秒数
template <typename Rep, typename Period,
          FMT_ENABLE_IF(std::is_integral<Rep>::value)>
inline std::chrono::duration<Rep, std::milli> get_milliseconds(
    std::chrono::duration<Rep, Period> d) {
  // 这可能会溢出，或者结果可能不适合目标类型
  // 返回时间间隔的毫秒数
#if FMT_SAFE_DURATION_CAST
  // 如果定义了 FMT_SAFE_DURATION_CAST，则使用通用秒类型
  using CommonSecondsType =
      typename std::common_type<decltype(d), std::chrono::seconds>::type;
  // 将输入的时间转换为通用秒类型
  const auto d_as_common = fmt_safe_duration_cast<CommonSecondsType>(d);
  // 将通用秒类型转换为整数秒
  const auto d_as_whole_seconds =
      fmt_safe_duration_cast<std::chrono::seconds>(d_as_common);
  // 计算时间的小数部分
  const auto diff = d_as_common - d_as_whole_seconds;
  // 将小数部分转换为毫秒
  const auto ms =
      fmt_safe_duration_cast<std::chrono::duration<Rep, std::milli>>(diff);
  // 返回毫秒
  return ms;
#else
  // 如果未定义 FMT_SAFE_DURATION_CAST，则直接将时间转换为秒
  auto s = std::chrono::duration_cast<std::chrono::seconds>(d);
  // 返回时间的毫秒部分
  return std::chrono::duration_cast<std::chrono::milliseconds>(d - s);
#endif
}

template <typename Rep, typename Period,
          FMT_ENABLE_IF(std::is_floating_point<Rep>::value)>
inline std::chrono::duration<Rep, std::milli> get_milliseconds(
    std::chrono::duration<Rep, Period> d) {
  // 定义通用类型为输入类型和整数类型的公共类型
  using common_type = typename std::common_type<Rep, std::intmax_t>::type;
  // 计算毫秒数
  auto ms = mod(d.count() * static_cast<common_type>(Period::num) /
                    static_cast<common_type>(Period::den) * 1000,
                1000);
  // 返回毫秒类型的时间
  return std::chrono::duration<Rep, std::milli>(static_cast<Rep>(ms));
}

template <typename Char, typename Rep, typename OutputIt>
OutputIt format_duration_value(OutputIt out, Rep val, int precision) {
  // 定义浮点数格式化字符串
  const Char pr_f[] = {'{', ':', '.', '{', '}', 'f', '}', 0};
  // 如果精度大于等于0，则使用指定精度格式化浮点数
  if (precision >= 0) return format_to(out, pr_f, val, precision);
  // 否则使用通用格式化字符串
  const Char fp_f[] = {'{', ':', 'g', '}', 0};
  const Char format[] = {'{', '}', 0};
  // 根据是否为浮点数类型选择格式化字符串格式化数值
  return format_to(out, std::is_floating_point<Rep>::value ? fp_f : format,
                   val);
}
template <typename Char, typename OutputIt>
OutputIt copy_unit(string_view unit, OutputIt out, Char) {
  // 将单位字符串复制到输出迭代器中
  return std::copy(unit.begin(), unit.end(), out);
}

template <typename OutputIt>
OutputIt copy_unit(string_view unit, OutputIt out, wchar_t) {
  // 将 UTF-8 编码的单元转换为 UTF-16 编码，并复制到输出迭代器中
  // 当 wchar_t 是 UTF-32 时，这个函数可以正常工作，因为单元只包含在 UTF-16 和 UTF-32 中具有相同表示的字符
  utf8_to_utf16 u(unit);
  return std::copy(u.c_str(), u.c_str() + u.size(), out);
}

template <typename Char, typename Period, typename OutputIt>
OutputIt format_duration_unit(OutputIt out) {
  // 如果 Period 对应的单位存在，则将其复制到输出迭代器中
  if (const char* unit = get_units<Period>())
    return copy_unit(string_view(unit), out, Char());
  // 如果 Period 对应的单位不存在，则根据 Period::num 的值格式化输出
  const Char num_f[] = {'[', '{', '}', ']', 's', 0};
  if (const_check(Period::den == 1)) return format_to(out, num_f, Period::num);
  // 如果 Period::den 不等于 1，则根据 Period::num 和 Period::den 的值格式化输出
  const Char num_def_f[] = {'[', '{', '}', '/', '{', '}', ']', 's', 0};
  return format_to(out, num_def_f, Period::num, Period::den);
}

template <typename FormatContext, typename OutputIt, typename Rep,
          typename Period>
struct chrono_formatter {
  FormatContext& context;
  OutputIt out;
  int precision;
  // rep is unsigned to avoid overflow.
  // rep 是无符号的，以避免溢出
  using rep =
      conditional_t<std::is_integral<Rep>::value && sizeof(Rep) < sizeof(int),
                    unsigned, typename make_unsigned_or_unchanged<Rep>::type>;
  rep val;
  using seconds = std::chrono::duration<rep>;
  seconds s;
  using milliseconds = std::chrono::duration<rep, std::milli>;
  bool negative;

  using char_type = typename FormatContext::char_type;

  // 构造函数，初始化成员变量
  explicit chrono_formatter(FormatContext& ctx, OutputIt o,
                            std::chrono::duration<Rep, Period> d)
      : context(ctx),
        out(o),
        val(static_cast<rep>(d.count())),
        negative(false) {
    // 如果 d 的值小于 0，则将 val 取反，并将 negative 置为 true
    if (d.count() < 0) {
      val = 0 - val;
      negative = true;
    }

    // 这可能会溢出，或者结果可能不适合目标类型
    // 可能需要检查转换（rep!=Rep）
#if FMT_SAFE_DURATION_CAST
    // 可能需要检查转换（rep!=Rep），将 val 转换为 seconds 类型
    auto tmpval = std::chrono::duration<rep, Period>(val);
    s = fmt_safe_duration_cast<seconds>(tmpval);
#else
    # 使用std::chrono命名空间中的duration_cast函数将时间段转换为秒
    s = std::chrono::duration_cast<seconds>(
        # 使用std::chrono命名空间中的duration函数创建一个时间段
        std::chrono::duration<rep, Period>(val));
#endif
  }

  // 如果值为 nan 或 inf，则返回 true，并写入输出
  bool handle_nan_inf() {
    // 如果值为有限数，则返回 false
    if (isfinite(val)) {
      return false;
    }
    // 如果值为 nan，则写入 nan 并返回 true
    if (isnan(val)) {
      write_nan();
      return true;
    }
    // 必须是 +-inf
    if (val > 0) {
      write_pinf();
    } else {
      write_ninf();
    }
    return true;
  }

  // 返回小时数
  Rep hour() const { return static_cast<Rep>(mod((s.count() / 3600), 24)); }

  // 返回 12 小时制的小时数
  Rep hour12() const {
    Rep hour = static_cast<Rep>(mod((s.count() / 3600), 12));
    return hour <= 0 ? 12 : hour;
  }

  // 返回分钟数
  Rep minute() const { return static_cast<Rep>(mod((s.count() / 60), 60)); }
  // 返回秒数
  Rep second() const { return static_cast<Rep>(mod(s.count(), 60)); }

  // 返回时间结构体
  std::tm time() const {
    auto time = std::tm();
    time.tm_hour = to_nonnegative_int(hour(), 24);
    time.tm_min = to_nonnegative_int(minute(), 60);
    time.tm_sec = to_nonnegative_int(second(), 60);
    return time;
  }

  // 写入符号
  void write_sign() {
    if (negative) {
      *out++ = '-';
      negative = false;
    }
  }

  // 写入值并指定宽度
  void write(Rep value, int width) {
    write_sign();
    if (isnan(value)) return write_nan();
    uint32_or_64_or_128_t<int> n =
        to_unsigned(to_nonnegative_int(value, max_value<int>()));
    int num_digits = detail::count_digits(n);
    if (width > num_digits) out = std::fill_n(out, width - num_digits, '0');
    out = format_decimal<char_type>(out, n, num_digits).end;
  }

  // 写入 nan
  void write_nan() { std::copy_n("nan", 3, out); }
  // 写入正无穷
  void write_pinf() { std::copy_n("inf", 3, out); }
  // 写入负无穷
  void write_ninf() { std::copy_n("-inf", 4, out); }

  // 格式化本地化时间
  void format_localized(const tm& time, char format, char modifier = 0) {
    if (isnan(val)) return write_nan();
    auto locale = context.locale().template get<std::locale>();
    auto& facet = std::use_facet<std::time_put<char_type>>(locale);
    std::basic_ostringstream<char_type> os;
    os.imbue(locale);
    facet.put(os, os, ' ', &time, format, modifier);
    auto str = os.str();
  }
  // 将输入字符串的内容复制到输出流中
  std::copy(str.begin(), str.end(), out);
  }

  // 处理文本内容，将指定范围内的字符复制到输出流中
  void on_text(const char_type* begin, const char_type* end) {
    std::copy(begin, end, out);
  }

  // 以下函数未实现，因为持续时间没有日期信息。
  void on_abbr_weekday() {}
  void on_full_weekday() {}
  void on_dec0_weekday(numeric_system) {}
  void on_dec1_weekday(numeric_system) {}
  void on_abbr_month() {}
  void on_full_month() {}
  void on_datetime(numeric_system) {}
  void on_loc_date(numeric_system) {}
  void on_loc_time(numeric_system) {}
  void on_us_date() {}
  void on_iso_date() {}
  void on_utc_offset() {}
  void on_tz_name() {}

  // 处理24小时制时间，根据数字系统写入小时数到输出流中
  void on_24_hour(numeric_system ns) {
    // 处理 NaN 和 Inf 的情况
    if (handle_nan_inf()) return;

    // 如果数字系统是标准的，直接写入小时数到输出流中
    if (ns == numeric_system::standard) return write(hour(), 2);
    auto time = tm();
    time.tm_hour = to_nonnegative_int(hour(), 24);
    format_localized(time, 'H', 'O');
  }

  // 处理12小时制时间，根据数字系统写入12小时制的小时数到输出流中
  void on_12_hour(numeric_system ns) {
    // 处理 NaN 和 Inf 的情况
    if (handle_nan_inf()) return;

    // 如果数字系统是标准的，直接写入12小时制的小时数到输出流中
    if (ns == numeric_system::standard) return write(hour12(), 2);
    auto time = tm();
    time.tm_hour = to_nonnegative_int(hour12(), 12);
    format_localized(time, 'I', 'O');
  }

  // 处理分钟数，根据数字系统写入分钟数到输出流中
  void on_minute(numeric_system ns) {
    // 处理 NaN 和 Inf 的情况
    if (handle_nan_inf()) return;

    // 如果数字系统是标准的，直接写入分钟数到输出流中
    if (ns == numeric_system::standard) return write(minute(), 2);
    auto time = tm();
    time.tm_min = to_nonnegative_int(minute(), 60);
    format_localized(time, 'M', 'O');
  }

  // 处理秒数，根据数字系统写入秒数到输出流中
  void on_second(numeric_system ns) {
    // 处理 NaN 和 Inf 的情况
    if (handle_nan_inf()) return;

    // 如果数字系统是标准的，直接写入秒数到输出流中
    write(second(), 2);
#if FMT_SAFE_DURATION_CAST
      // 如果启用了安全的持续时间转换，则将 rep 转换为 Rep
      using duration_rep = std::chrono::duration<rep, Period>;
      using duration_Rep = std::chrono::duration<Rep, Period>;
      // 使用 fmt_safe_duration_cast 将 duration_rep 转换为 duration_Rep
      auto tmpval = fmt_safe_duration_cast<duration_Rep>(duration_rep{val});
#else
      // 否则直接将 val 赋给 tmpval
      auto tmpval = std::chrono::duration<Rep, Period>(val);
#endif
      // 获取 tmpval 的毫秒数
      auto ms = get_milliseconds(tmpval);
      // 如果毫秒数不为 0，则在输出中添加小数点和毫秒数
      if (ms != std::chrono::milliseconds(0)) {
        *out++ = '.';
        write(ms.count(), 3);
      }
      // 返回
      return;
    }
    // 创建 time 对象
    auto time = tm();
    // 设置 time 对象的秒数为非负整数
    time.tm_sec = to_nonnegative_int(second(), 60);
    // 格式化输出 time 对象的秒和时区偏移
    format_localized(time, 'S', 'O');
  }

  // 处理 12 小时制时间
  void on_12_hour_time() {
    // 如果处理 NaN 或 Inf，则返回
    if (handle_nan_inf()) return;
    // 格式化输出时间，使用 'r' 标志
    format_localized(time(), 'r');
  }

  // 处理 24 小时制时间
  void on_24_hour_time() {
    // 如果处理 NaN 或 Inf，则在输出中添加冒号并返回
    if (handle_nan_inf()) {
      *out++ = ':';
      handle_nan_inf();
      return;
    }
    // 写入小时数并在输出中添加冒号
    write(hour(), 2);
    *out++ = ':';
    // 写入分钟数
    write(minute(), 2);
  }

  // 处理 ISO 时间
  void on_iso_time() {
    // 处理 24 小时制时间
    on_24_hour_time();
    *out++ = ':';
    // 如果处理 NaN 或 Inf，则返回
    if (handle_nan_inf()) return;
    // 写入秒数
    write(second(), 2);
  }

  // 处理上午下午时间
  void on_am_pm() {
    // 如果处理 NaN 或 Inf，则返回
    if (handle_nan_inf()) return;
    // 格式化输出时间，使用 'p' 标志
    format_localized(time(), 'p');
  }

  // 处理持续时间值
  void on_duration_value() {
    // 如果处理 NaN 或 Inf，则返回
    if (handle_nan_inf()) return;
    // 写入符号
    write_sign();
    // 格式化输出持续时间值
    out = format_duration_value<char_type>(out, val, precision);
  }

  // 处理持续时间单位
  void on_duration_unit() {
    // 格式化输出持续时间单位
    out = format_duration_unit<char_type, Period>(out);
  }
};
}  // namespace detail

// 模板特化，用于格式化 std::chrono::duration 类型的参数
template <typename Rep, typename Period, typename Char>
struct formatter<std::chrono::duration<Rep, Period>, Char> {
  private:
  // 基本格式规范
  basic_format_specs<Char> specs;
  // 精度
  int precision;
  // 参数引用类型
  using arg_ref_type = detail::arg_ref<Char>;
  // 宽度引用
  arg_ref_type width_ref;
  // 精度引用
  arg_ref_type precision_ref;
  // 可变的格式字符串视图
  mutable basic_string_view<Char> format_str;
  // 持续时间类型
  using duration = std::chrono::duration<Rep, Period>;

  // 规范处理器
  struct spec_handler {
    // 格式化器
    formatter& f;
    // 基本格式解析上下文
    basic_format_parse_context<Char>& context;
    // 格式字符串视图
    // 创建一个引用参数，用于检查参数ID是否有效
    template <typename Id> FMT_CONSTEXPR arg_ref_type make_arg_ref(Id arg_id) {
      context.check_arg_id(arg_id);
      return arg_ref_type(arg_id);
    }

    // 创建一个引用参数，用于检查字符串参数ID是否有效
    FMT_CONSTEXPR arg_ref_type make_arg_ref(basic_string_view<Char> arg_id) {
      context.check_arg_id(arg_id);
      return arg_ref_type(arg_id);
    }

    // 创建一个引用参数，用于自动生成参数ID
    FMT_CONSTEXPR arg_ref_type make_arg_ref(detail::auto_id) {
      return arg_ref_type(context.next_arg_id());
    }

    // 当出现错误时抛出异常
    void on_error(const char* msg) { FMT_THROW(format_error(msg)); }
    // 设置填充字符
    void on_fill(basic_string_view<Char> fill) { f.specs.fill = fill; }
    // 设置对齐方式
    void on_align(align_t align) { f.specs.align = align; }
    // 设置宽度
    void on_width(int width) { f.specs.width = width; }
    // 设置精度
    void on_precision(int _precision) { f.precision = _precision; }
    // 结束精度设置
    void end_precision() {}

    // 设置动态宽度
    template <typename Id> void on_dynamic_width(Id arg_id) {
      f.width_ref = make_arg_ref(arg_id);
    }

    // 设置动态精度
    template <typename Id> void on_dynamic_precision(Id arg_id) {
      f.precision_ref = make_arg_ref(arg_id);
    }
  };

  // 定义迭代器类型
  using iterator = typename basic_format_parse_context<Char>::iterator;
  // 定义解析范围结构体
  struct parse_range {
    iterator begin;
    iterator end;
  };

  // 执行解析操作
  FMT_CONSTEXPR parse_range do_parse(basic_format_parse_context<Char>& ctx) {
    auto begin = ctx.begin(), end = ctx.end();
    // 如果开始迭代器等于结束迭代器或者开始迭代器指向'}'，则返回空范围
    if (begin == end || *begin == '}') return {begin, begin};
    // 创建格式处理器对象
    spec_handler handler{*this, ctx, format_str};
    // 解析对齐方式
    begin = detail::parse_align(begin, end, handler);
    // 如果已经到达结束，则返回空范围
    if (begin == end) return {begin, begin};
    // 解析宽度
    begin = detail::parse_width(begin, end, handler);
    // 如果已经到达结束，则返回空范围
    if (begin == end) return {begin, begin};
    // 如果下一个字符是'.'，则解析精度
    if (*begin == '.') {
      // 如果Rep是浮点类型，则解析精度
      if (std::is_floating_point<Rep>::value)
        begin = detail::parse_precision(begin, end, handler);
      // 否则报错，该类型不支持精度设置
      else
        handler.on_error("precision not allowed for this argument type");
    }
    // 解析时间格式
    end = parse_chrono_format(begin, end, detail::chrono_format_checker());
    // 返回一个包含begin和end的花括号初始化列表
    return {begin, end};
  }

 public:
  // 默认构造函数，初始化precision为-1
  formatter() : precision(-1) {}

  // 解析格式字符串
  FMT_CONSTEXPR auto parse(basic_format_parse_context<Char>& ctx)
      -> decltype(ctx.begin()) {
    // 执行解析操作
    auto range = do_parse(ctx);
    // 将解析得到的范围转换为basic_string_view
    format_str = basic_string_view<Char>(
        &*range.begin, detail::to_unsigned(range.end - range.begin));
    // 返回解析结束的位置
    return range.end;
  }

  // 格式化duration类型的数据
  template <typename FormatContext>
  auto format(const duration& d, FormatContext& ctx) -> decltype(ctx.out()) {
    auto begin = format_str.begin(), end = format_str.end();
    // 作为可能的未来优化，如果未指定宽度，可以避免额外的复制
    basic_memory_buffer<Char> buf;
    auto out = std::back_inserter(buf);
    // 处理动态规格
    detail::handle_dynamic_spec<detail::width_checker>(specs.width, width_ref,
                                                       ctx);
    detail::handle_dynamic_spec<detail::precision_checker>(precision,
                                                           precision_ref, ctx);
    // 如果begin等于end或者begin指向'}'，则格式化duration值和单位
    if (begin == end || *begin == '}') {
      out = detail::format_duration_value<Char>(out, d.count(), precision);
      detail::format_duration_unit<Char, Period>(out);
    } else {
      // 创建chrono_formatter对象，用于格式化duration
      detail::chrono_formatter<FormatContext, decltype(out), Rep, Period> f(
          ctx, out, d);
      f.precision = precision;
      // 解析chrono格式
      parse_chrono_format(begin, end, f);
    }
    // 将格式化后的数据写入输出流
    return detail::write(
        ctx.out(), basic_string_view<Char>(buf.data(), buf.size()), specs);
  }
};

FMT_END_NAMESPACE

#endif  // FMT_CHRONO_H_
```