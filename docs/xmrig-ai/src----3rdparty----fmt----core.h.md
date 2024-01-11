# `xmrig\src\3rdparty\fmt\core.h`

```
// 格式化 C++ 的库 - 核心 API
//
// 版权所有，Victor Zverovich，2012年至今
//
// 有关许可信息，请参阅 format.h。

#ifndef FMT_CORE_H_
#define FMT_CORE_H_

#include <cstdio>  // std::FILE
#include <cstring>
#include <functional>
#include <iterator>
#include <memory>
#include <string>
#include <type_traits>
#include <vector>

// fmt 库的版本，形式为 major * 10000 + minor * 100 + patch。
#define FMT_VERSION 70003

#ifdef __clang__
#  define FMT_CLANG_VERSION (__clang_major__ * 100 + __clang_minor__)
#else
#  define FMT_CLANG_VERSION 0
#endif

#if defined(__GNUC__) && !defined(__clang__)
#  define FMT_GCC_VERSION (__GNUC__ * 100 + __GNUC_MINOR__)
#else
#  define FMT_GCC_VERSION 0
#endif

#if defined(__INTEL_COMPILER)
#  define FMT_ICC_VERSION __INTEL_COMPILER
#else
#  define FMT_ICC_VERSION 0
#endif

#if __cplusplus >= 201103L || defined(__GXX_EXPERIMENTAL_CXX0X__)
#  define FMT_HAS_GXX_CXX11 FMT_GCC_VERSION
#else
#  define FMT_HAS_GXX_CXX11 0
#endif

#ifdef __NVCC__
#  define FMT_NVCC __NVCC__
#else
#  define FMT_NVCC 0
#endif

#ifdef _MSC_VER
#  define FMT_MSC_VER _MSC_VER
#  define FMT_SUPPRESS_MSC_WARNING(n) __pragma(warning(suppress : n))
#else
#  define FMT_MSC_VER 0
#  define FMT_SUPPRESS_MSC_WARNING(n)
#endif

#ifdef __has_feature
#  define FMT_HAS_FEATURE(x) __has_feature(x)
#else
#  define FMT_HAS_FEATURE(x) 0
#endif

#if defined(__has_include) && !defined(__INTELLISENSE__) && \
    (!FMT_ICC_VERSION || FMT_ICC_VERSION >= 1600)
#  define FMT_HAS_INCLUDE(x) __has_include(x)
#else
#  define FMT_HAS_INCLUDE(x) 0
#endif

#ifdef __has_cpp_attribute
#  define FMT_HAS_CPP_ATTRIBUTE(x) __has_cpp_attribute(x)
#else
#  define FMT_HAS_CPP_ATTRIBUTE(x) 0
#endif

#define FMT_HAS_CPP14_ATTRIBUTE(attribute) \
  (__cplusplus >= 201402L && FMT_HAS_CPP_ATTRIBUTE(attribute))

#define FMT_HAS_CPP17_ATTRIBUTE(attribute) \
  (__cplusplus >= 201703L && FMT_HAS_CPP_ATTRIBUTE(attribute))
// 检查是否支持放松的 C++14 constexpr。
// GCC 在版本 6 之前不允许在 constexpr 中使用 throw（bug 67371）。
#ifndef FMT_USE_CONSTEXPR
#  define FMT_USE_CONSTEXPR                                           \
    (FMT_HAS_FEATURE(cxx_relaxed_constexpr) || FMT_MSC_VER >= 1910 || \
     (FMT_GCC_VERSION >= 600 && __cplusplus >= 201402L)) &&           \
        !FMT_NVCC && !FMT_ICC_VERSION
#endif
#if FMT_USE_CONSTEXPR
#  define FMT_CONSTEXPR constexpr
#  define FMT_CONSTEXPR_DECL constexpr
#else
#  define FMT_CONSTEXPR inline
#  define FMT_CONSTEXPR_DECL
#endif

#ifndef FMT_OVERRIDE
#  if FMT_HAS_FEATURE(cxx_override_control) || \
      (FMT_GCC_VERSION >= 408 && FMT_HAS_GXX_CXX11) || FMT_MSC_VER >= 1900
#    define FMT_OVERRIDE override
#  else
#    define FMT_OVERRIDE
#  endif
#endif

// 检查是否禁用了异常。
#ifndef FMT_EXCEPTIONS
#  if (defined(__GNUC__) && !defined(__EXCEPTIONS)) || \
      FMT_MSC_VER && !_HAS_EXCEPTIONS
#    define FMT_EXCEPTIONS 0
#  else
#    define FMT_EXCEPTIONS 1
#  endif
#endif

// 定义 FMT_USE_NOEXCEPT 以使 fmt 使用 noexcept（C++11 特性）。
#ifndef FMT_USE_NOEXCEPT
#  define FMT_USE_NOEXCEPT 0
#endif

#if FMT_USE_NOEXCEPT || FMT_HAS_FEATURE(cxx_noexcept) || \
    (FMT_GCC_VERSION >= 408 && FMT_HAS_GXX_CXX11) || FMT_MSC_VER >= 1900
#  define FMT_DETECTED_NOEXCEPT noexcept
#  define FMT_HAS_CXX11_NOEXCEPT 1
#else
#  define FMT_DETECTED_NOEXCEPT throw()
#  define FMT_HAS_CXX11_NOEXCEPT 0
#endif

#ifndef FMT_NOEXCEPT
#  if FMT_EXCEPTIONS || FMT_HAS_CXX11_NOEXCEPT
#    define FMT_NOEXCEPT FMT_DETECTED_NOEXCEPT
#  else
#    define FMT_NOEXCEPT
#  endif
#endif

// [[noreturn]] 在 MSVC 和 NVCC 上被禁用，因为会产生虚假的不可达代码警告。
#if FMT_EXCEPTIONS && FMT_HAS_CPP_ATTRIBUTE(noreturn) && !FMT_MSC_VER && \
    !FMT_NVCC
#  define FMT_NORETURN [[noreturn]]
#else
#  define FMT_NORETURN
#endif

#ifndef FMT_DEPRECATED
#  if FMT_HAS_CPP14_ATTRIBUTE(deprecated) || FMT_MSC_VER >= 1900
// 定义 FMT_DEPRECATED 宏，标记为已废弃
#define FMT_DEPRECATED [[deprecated]]
// 如果不是 C++17 及以上版本，则根据不同编译器定义 FMT_DEPRECATED 宏
#else
// 如果是 GCC 或者 Clang 编译器，则使用 __attribute__((deprecated)) 标记为已废弃
if (defined(__GNUC__) && !defined(__LCC__)) || defined(__clang__)
#define FMT_DEPRECATED __attribute__((deprecated))
// 如果是 MSC 编译器，则使用 __declspec(deprecated) 标记为已废弃
elif FMT_MSC_VER
#define FMT_DEPRECATED __declspec(deprecated)
// 否则，定义 FMT_DEPRECATED 为空
else
#define FMT_DEPRECATED /* deprecated */
#endif
#endif

// 解决 Intel、PGI 和 NVCC 编译器中的 [[deprecated]] 问题
#if FMT_ICC_VERSION || defined(__PGI) || FMT_NVCC
// 定义 FMT_DEPRECATED_ALIAS 为空
#define FMT_DEPRECATED_ALIAS
// 否则，定义 FMT_DEPRECATED_ALIAS 为 FMT_DEPRECATED
#else
#define FMT_DEPRECATED_ALIAS FMT_DEPRECATED
#endif

// 如果未定义 FMT_INLINE
#ifndef FMT_INLINE
// 如果是 GCC 或者 Clang 编译器，则定义 FMT_INLINE 为 inline __attribute__((always_inline))
#if FMT_GCC_VERSION || FMT_CLANG_VERSION
#define FMT_INLINE inline __attribute__((always_inline))
// 否则，定义 FMT_INLINE 为 inline
#else
#define FMT_INLINE inline
#endif
#endif

// 如果未定义 FMT_BEGIN_NAMESPACE
#ifndef FMT_BEGIN_NAMESPACE
// 如果支持 cxx_inline_namespaces 特性，或者是 GCC 版本大于等于 404，或者是 MSC 版本大于等于 1900 且不是 _MANAGED 环境
#if FMT_HAS_FEATURE(cxx_inline_namespaces) || FMT_GCC_VERSION >= 404 || \
    (FMT_MSC_VER >= 1900 && !_MANAGED)
// 定义 FMT_INLINE_NAMESPACE 为 inline namespace，定义 FMT_END_NAMESPACE 为结束命名空间
#define FMT_INLINE_NAMESPACE inline namespace
#define FMT_END_NAMESPACE \
  }                       \
  }
// 否则，定义 FMT_INLINE_NAMESPACE 为命名空间，定义 FMT_END_NAMESPACE 为结束命名空间并使用 v7 命名空间
#else
#define FMT_INLINE_NAMESPACE namespace
#define FMT_END_NAMESPACE \
  }                       \
  using namespace v7;     \
  }
#endif
// 定义 FMT_BEGIN_NAMESPACE 为 fmt 命名空间
#define FMT_BEGIN_NAMESPACE \
namespace fmt {           \
FMT_INLINE_NAMESPACE v7 {
#endif

// 如果未定义 FMT_HEADER_ONLY 且在 _WIN32 环境下
#if !defined(FMT_HEADER_ONLY) && defined(_WIN32)
// 定义 FMT_CLASS_API 为 FMT_SUPPRESS_MSC_WARNING(4275)
#define FMT_CLASS_API FMT_SUPPRESS_MSC_WARNING(4275)
// 如果定义了 FMT_EXPORT
#ifdef FMT_EXPORT
// 定义 FMT_API 为 __declspec(dllexport)，定义 FMT_EXTERN_TEMPLATE_API 为 FMT_API，定义 FMT_EXPORTED
#define FMT_API __declspec(dllexport)
#define FMT_EXTERN_TEMPLATE_API FMT_API
#define FMT_EXPORTED
// 否则，如果定义了 FMT_SHARED
#elif defined(FMT_SHARED)
// 定义 FMT_API 为 __declspec(dllimport)，定义 FMT_EXTERN_TEMPLATE_API 为 FMT_API
#define FMT_API __declspec(dllimport)
#define FMT_EXTERN_TEMPLATE_API FMT_API
#endif
// 否则，定义 FMT_CLASS_API 为空
#else
#define FMT_CLASS_API
#endif
// 如果未定义 FMT_API，则定义 FMT_API 为空
#ifndef FMT_API
#define FMT_API
#endif
// 如果未定义 FMT_EXTERN_TEMPLATE_API，则定义 FMT_EXTERN_TEMPLATE_API 为空
#ifndef FMT_EXTERN_TEMPLATE_API
#define FMT_EXTERN_TEMPLATE_API
#endif
// 如果未定义 FMT_INSTANTIATION_DEF_API，则定义 FMT_INSTANTIATION_DEF_API 为 FMT_API
#ifndef FMT_INSTANTIATION_DEF_API
#define FMT_INSTANTIATION_DEF_API FMT_API
#endif

// 如果未定义 FMT_HEADER_ONLY，则定义 FMT_EXTERN 为 extern
#ifndef FMT_HEADER_ONLY
#define FMT_EXTERN extern
// 否则，定义 FMT_EXTERN 为空
#else
#define FMT_EXTERN
#endif

// libc++ 在 C++17 之前支持 string_view
# 如果编译器支持 <string_view> 并且 C++ 版本大于 201402L 或者定义了 _LIBCPP_VERSION，或者
# 如果定义了 _MSVC_LANG 并且 _MSVC_LANG 大于 201402L 并且 _MSC_VER 大于等于 1910
# 则包含 <string_view> 头文件，并定义 FMT_USE_STRING_VIEW
# 否则，如果编译器支持 "experimental/string_view" 并且 C++ 版本大于等于 201402L
# 则包含 <experimental/string_view> 头文件，并定义 FMT_USE_EXPERIMENTAL_STRING_VIEW
#ifndef FMT_UNICODE
#  如果未定义 FMT_UNICODE，则定义为非 FMT_MSC_VER
#endif
# 如果定义了 FMT_UNICODE 并且 FMT_MSC_VER
# 则设置执行字符集为 "utf-8"
FMT_BEGIN_NAMESPACE

// 为旧系统实现 enable_if_t 和其他元函数
template <bool B, class T = void>
using enable_if_t = typename std::enable_if<B, T>::type;
template <bool B, class T, class F>
using conditional_t = typename std::conditional<B, T, F>::type;
template <bool B> using bool_constant = std::integral_constant<bool, B>;
template <typename T>
using remove_reference_t = typename std::remove_reference<T>::type;
template <typename T>
using remove_const_t = typename std::remove_const<T>::type;
template <typename T>
using remove_cvref_t = typename std::remove_cv<remove_reference_t<T>>::type;
template <typename T> struct type_identity { using type = T; };
template <typename T> using type_identity_t = typename type_identity<T>::type;

struct monostate {};

// 用于在模板参数中使用的 enable_if 辅助宏，可以使符号更短
// 需要额外的括号来解决 MSVC 2019 中的一个 bug
#define FMT_ENABLE_IF(...) enable_if_t<(__VA_ARGS__), int> = 0

namespace detail {

// 一个用于抑制 "条件表达式是常量" 警告的辅助函数
template <typename T> constexpr T const_check(T value) { return value; }

// 断言失败时的处理函数
FMT_NORETURN FMT_API void assert_fail(const char* file, int line,
                                      const char* message);

#ifndef FMT_ASSERT
#  ifdef NDEBUG
// 定义 FMT_ASSERT 宏，用于断言条件是否成立，如果条件不成立则调用 assert_fail 函数
#    define FMT_ASSERT(condition, message) ((void)0)
#  else
#    define FMT_ASSERT(condition, message)                                    \
      ((condition) /* void() fails with -Winvalid-constexpr on clang 4.0.1 */ \
           ? (void)0                                                          \
           : ::fmt::detail::assert_fail(__FILE__, __LINE__, (message)))
#  endif
#endif

// 根据宏定义选择使用标准库的 string_view 还是实验性的 string_view
#if defined(FMT_USE_STRING_VIEW)
template <typename Char> using std_string_view = std::basic_string_view<Char>;
#elif defined(FMT_USE_EXPERIMENTAL_STRING_VIEW)
template <typename Char>
using std_string_view = std::experimental::basic_string_view<Char>;
#else
template <typename T> struct std_string_view {};
#endif

// 根据宏定义选择是否使用 int128 类型
#ifdef FMT_USE_INT128
// Do nothing.
#elif defined(__SIZEOF_INT128__) && !FMT_NVCC && \
    !(FMT_CLANG_VERSION && FMT_MSC_VER)
#  define FMT_USE_INT128 1
using int128_t = __int128_t;
using uint128_t = __uint128_t;
#else
#  define FMT_USE_INT128 0
#endif
#if !FMT_USE_INT128
struct int128_t {};
struct uint128_t {};
#endif

// 将非负整数转换为无符号整数
template <typename Int>
FMT_CONSTEXPR typename std::make_unsigned<Int>::type to_unsigned(Int value) {
  FMT_ASSERT(value >= 0, "negative value");
  return static_cast<typename std::make_unsigned<Int>::type>(value);
}

// 定义一个包含 Unicode 字符的数组 micro
FMT_SUPPRESS_MSC_WARNING(4566) constexpr unsigned char micro[] = "\u00B5";

// 检查是否为 Unicode 字符
template <typename Char> constexpr bool is_unicode() {
  return FMT_UNICODE || sizeof(Char) != 1 ||
         (sizeof(micro) == 3 && micro[0] == 0xC2 && micro[1] == 0xB5);
}

// 根据宏定义选择是否使用 char8_t 类型
#ifdef __cpp_char8_t
using char8_type = char8_t;
#else
enum char8_type : unsigned char {};
#endif
}  // namespace detail

// 根据宏定义选择是否使用内部命名空间
#ifdef FMT_USE_INTERNAL
namespace internal = detail;  // DEPRECATED
#endif
/**
  一个用于 pre-C++17 的 ``std::basic_string_view`` 的实现。它提供了 API 的一个子集。
  即使 ``std::string_view`` 可用，``fmt::basic_string_view`` 也用于格式化字符串，以防止在库使用不同的 ``-std`` 选项编译时与客户端代码出现问题（这是不推荐的）。
 */
template <typename Char> class basic_string_view {
 private:
  const Char* data_;  // 字符串数据的指针
  size_t size_;  // 字符串的大小

 public:
  using value_type = Char;  // 字符类型
  using iterator = const Char*;  // 迭代器类型

  constexpr basic_string_view() FMT_NOEXCEPT : data_(nullptr), size_(0) {}  // 默认构造函数

  /** 从 C 字符串和大小构造一个字符串引用对象。 */
  constexpr basic_string_view(const Char* s, size_t count) FMT_NOEXCEPT
      : data_(s),
        size_(count) {}

  /**
    \rst
    从 C 字符串构造一个字符串引用对象，使用 ``std::char_traits<Char>::length`` 计算大小。
    \endrst
   */
#if __cplusplus >= 201703L  // C++17 的 char_traits::length() 是 constexpr。
  FMT_CONSTEXPR
#endif
// 从指针构造一个基本字符串视图
basic_string_view(const Char* s)
    : data_(s), size_(std::char_traits<Char>::length(s)) {}

/** 从``std::basic_string``对象构造一个字符串引用。 */
template <typename Traits, typename Alloc>
FMT_CONSTEXPR basic_string_view(
    const std::basic_string<Char, Traits, Alloc>& s) FMT_NOEXCEPT
    : data_(s.data()),
      size_(s.size()) {}

template <typename S, FMT_ENABLE_IF(std::is_same<
                                  S, detail::std_string_view<Char>>::value)>
FMT_CONSTEXPR basic_string_view(S s) FMT_NOEXCEPT : data_(s.data()),
                                                  size_(s.size()) {}

/** 返回指向字符串数据的指针。 */
constexpr const Char* data() const { return data_; }

/** 返回字符串大小。 */
constexpr size_t size() const { return size_; }

constexpr iterator begin() const { return data_; }
constexpr iterator end() const { return data_ + size_; }

constexpr const Char& operator[](size_t pos) const { return data_[pos]; }

FMT_CONSTEXPR void remove_prefix(size_t n) {
  data_ += n;
  size_ -= n;
}

// 按字典顺序比较此字符串引用和其他字符串引用。
int compare(basic_string_view other) const {
  size_t str_size = size_ < other.size_ ? size_ : other.size_;
  int result = std::char_traits<Char>::compare(data_, other.data_, str_size);
  if (result == 0)
    result = size_ == other.size_ ? 0 : (size_ < other.size_ ? -1 : 1);
  return result;
}

friend bool operator==(basic_string_view lhs, basic_string_view rhs) {
  return lhs.compare(rhs) == 0;
}
friend bool operator!=(basic_string_view lhs, basic_string_view rhs) {
  return lhs.compare(rhs) != 0;
}
friend bool operator<(basic_string_view lhs, basic_string_view rhs) {
  return lhs.compare(rhs) < 0;
}
friend bool operator<=(basic_string_view lhs, basic_string_view rhs) {
    # 比较两个字符串视图的大小，如果左边小于等于右边，则返回 true
    return lhs.compare(rhs) <= 0;
  }
  # 重载操作符，比较两个字符串视图的大小，如果左边大于右边，则返回 true
  friend bool operator>(basic_string_view lhs, basic_string_view rhs) {
    return lhs.compare(rhs) > 0;
  }
  # 重载操作符，比较两个字符串视图的大小，如果左边大于等于右边，则返回 true
  friend bool operator>=(basic_string_view lhs, basic_string_view rhs) {
    return lhs.compare(rhs) >= 0;
  }
};

// 使用别名定义 string_view 和 wstring_view
using string_view = basic_string_view<char>;
using wstring_view = basic_string_view<wchar_t>;

/** Specifies if ``T`` is a character type. Can be specialized by users. */
// 模板结构体，用于判断是否为字符类型
template <typename T> struct is_char : std::false_type {};
template <> struct is_char<char> : std::true_type {};
template <> struct is_char<wchar_t> : std::true_type {};
template <> struct is_char<detail::char8_type> : std::true_type {};
template <> struct is_char<char16_t> : std::true_type {};
template <> struct is_char<char32_t> : std::true_type {};

/**
  \rst
  Returns a string view of `s`. In order to add custom string type support to
  {fmt} provide an overload of `to_string_view` for it in the same namespace as
  the type for the argument-dependent lookup to work.

  **Example**::

    namespace my_ns {
    inline string_view to_string_view(const my_string& s) {
      return {s.data(), s.length()};
    }
    }
    std::string message = fmt::format(my_string("The answer is {}"), 42);
  \endrst
 */
// 返回字符串视图的模板函数，用于将不同类型的字符串转换为视图
template <typename Char, FMT_ENABLE_IF(is_char<Char>::value)>
inline basic_string_view<Char> to_string_view(const Char* s) {
  return s;
}

// 重载，用于将 std::basic_string 转换为视图
template <typename Char, typename Traits, typename Alloc>
inline basic_string_view<Char> to_string_view(
    const std::basic_string<Char, Traits, Alloc>& s) {
  return s;
}

// 重载，用于将 basic_string_view 转换为视图
template <typename Char>
inline basic_string_view<Char> to_string_view(basic_string_view<Char> s) {
  return s;
}

// 重载，用于将 detail::std_string_view 转换为视图
template <typename Char,
          FMT_ENABLE_IF(!std::is_empty<detail::std_string_view<Char>>::value)>
inline basic_string_view<Char> to_string_view(detail::std_string_view<Char> s) {
  return s;
}

// 用于编译时字符串的基类，定义在 fmt 命名空间中，以便通过 ADL 可见格式化函数
// 例如 format(FMT_STRING("{}"), 42)
struct compile_string {};

// 判断是否为编译时字符串的模板结构体
template <typename S>
struct is_compile_string : std::is_base_of<compile_string, S> {};

// 用于编译时字符串的模板函数
template <typename S, FMT_ENABLE_IF(is_compile_string<S>::value)>
// 将字符串类型转换为基本的字符串视图
constexpr basic_string_view<typename S::char_type> to_string_view(const S& s) {
  return s;
}

namespace detail {
// 重载函数，用于处理其他类型的参数
void to_string_view(...);
// 使用 fmt::v7::to_string_view 命名空间
using fmt::v7::to_string_view;

// 指定 S 是否是可转换为 fmt::basic_string_view 的字符串类型
// 应该是一个 constexpr 函数，但 MSVC 2017 无法在 enable_if 中编译它
// MSVC 2015 无法将其编译为别名模板
template <typename S>
struct is_string : std::is_class<decltype(to_string_view(std::declval<S>()))> {
};

template <typename S, typename = void> struct char_t_impl {};
// 如果 S 是字符串类型，则使用 enable_if
template <typename S> struct char_t_impl<S, enable_if_t<is_string<S>::value>> {
  using result = decltype(to_string_view(std::declval<S>()));
  using type = typename result::value_type;
};

// 如果 S 不是有效的格式字符串，则报告编译时错误
template <typename..., typename S, FMT_ENABLE_IF(!is_compile_string<S>::value)>
FMT_INLINE void check_format_string(const S&) {
#ifdef FMT_ENFORCE_COMPILE_STRING
  static_assert(is_compile_string<S>::value,
                "FMT_ENFORCE_COMPILE_STRING requires all format strings to use "
                "FMT_STRING.");
#endif
}
// 如果 S 是有效的格式字符串，则检查格式字符串
template <typename..., typename S, FMT_ENABLE_IF(is_compile_string<S>::value)>
void check_format_string(S);

// 错误处理程序
struct error_handler {
  constexpr error_handler() = default;
  constexpr error_handler(const error_handler&) = default;

  // 这个函数故意不是 constexpr 的，以便产生编译时错误
  FMT_NORETURN FMT_API void on_error(const char* message);
};
}  // namespace detail

/** 字符串的字符类型 */
template <typename S> using char_t = typename detail::char_t_impl<S>::type;
/**
  \rst
  Parsing context consisting of a format string range being parsed and an
  argument counter for automatic indexing.

  You can use one of the following type aliases for common character types:

  +-----------------------+-------------------------------------+
  | Type                  | Definition                          |
  +=======================+=====================================+
  | format_parse_context  | basic_format_parse_context<char>    |
  +-----------------------+-------------------------------------+
  | wformat_parse_context | basic_format_parse_context<wchar_t> |
  +-----------------------+-------------------------------------+
  \endrst
 */
template <typename Char, typename ErrorHandler = detail::error_handler>
class basic_format_parse_context : private ErrorHandler {
 private:
  basic_string_view<Char> format_str_;
  int next_arg_id_;

 public:
  using char_type = Char;
  using iterator = typename basic_string_view<Char>::iterator;

  // 构造函数，初始化格式字符串和下一个参数的索引
  explicit constexpr basic_format_parse_context(
      basic_string_view<Char> format_str, ErrorHandler eh = {},
      int next_arg_id = 0)
      : ErrorHandler(eh), format_str_(format_str), next_arg_id_(next_arg_id) {}

  /**
    Returns an iterator to the beginning of the format string range being
    parsed.
   */
  // 返回格式字符串被解析的起始迭代器
  constexpr iterator begin() const FMT_NOEXCEPT { return format_str_.begin(); }

  /**
    Returns an iterator past the end of the format string range being parsed.
   */
  // 返回格式字符串被解析的结束迭代器
  constexpr iterator end() const FMT_NOEXCEPT { return format_str_.end(); }

  /** Advances the begin iterator to ``it``. */
  // 将起始迭代器前进到指定位置
  FMT_CONSTEXPR void advance_to(iterator it) {
    format_str_.remove_prefix(detail::to_unsigned(it - begin()));
  }

  /**
    Reports an error if using the manual argument indexing; otherwise returns
    the next argument index and switches to the automatic indexing.
   */
  // 如果使用手动参数索引，则报告错误；否则返回下一个参数索引并切换到自动索引
  FMT_CONSTEXPR int next_arg_id() {
    // Don't check if the argument id is valid to avoid overhead and because it
  // 如果下一个参数的索引大于等于0，则返回下一个参数的索引并自增1
  if (next_arg_id_ >= 0) return next_arg_id_++;
  // 如果尝试从手动切换到自动参数索引，则报错
  on_error("cannot switch from manual to automatic argument indexing");
  // 返回0
  return 0;
}

/**
  如果使用自动参数索引，则报错；否则切换到手动索引
 */
FMT_CONSTEXPR void check_arg_id(int) {
  if (next_arg_id_ > 0)
    // 如果尝试从自动切换到手动参数索引，则报错
    on_error("cannot switch from automatic to manual argument indexing");
  else
    // 切换到手动参数索引
    next_arg_id_ = -1;
}

FMT_CONSTEXPR void check_arg_id(basic_string_view<Char>) {}

FMT_CONSTEXPR void on_error(const char* message) {
  // 调用错误处理器的错误处理方法
  ErrorHandler::on_error(message);
}

// 返回错误处理器
constexpr ErrorHandler error_handler() const { return *this; }
// 结束了一个代码块
};

// 使用别名定义格式解析上下文
using format_parse_context = basic_format_parse_context<char>;
// 使用别名定义宽字符格式解析上下文
using wformat_parse_context = basic_format_parse_context<wchar_t>;

// 模板类，用于表示格式化参数
template <typename Context> class basic_format_arg;
// 模板类，用于表示格式化参数列表
template <typename Context> class basic_format_args;
// 模板类，用于表示动态格式化参数存储
template <typename Context> class dynamic_format_arg_store;

// 用于对象类型 T 的格式化器
template <typename T, typename Char = char, typename Enable = void>
struct formatter {
  // 删除默认构造函数表示禁用的格式化器
  formatter() = delete;
};

// 指定类型 T 是否有启用的格式化器特化。即使类型没有格式化器，也可以是可格式化的，例如通过转换。
template <typename T, typename Context>
using has_formatter =
    std::is_constructible<typename Context::template formatter_type<T>>;

// 检查类型 T 是否是具有连续存储的容器
template <typename T> struct is_contiguous : std::false_type {};
template <typename Char>
struct is_contiguous<std::basic_string<Char>> : std::true_type {};

namespace detail {

// 从 back_insert_iterator 中提取对容器的引用
template <typename Container>
inline Container& get_container(std::back_insert_iterator<Container> it) {
  using bi_iterator = std::back_insert_iterator<Container>;
  struct accessor : bi_iterator {
    accessor(bi_iterator iter) : bi_iterator(iter) {}
    using bi_iterator::container;
  };
  return *accessor(it).container;
}

/**
  \rst
  一个具有可选增长能力的连续内存缓冲区。这是一个内部类，不应直接使用，只能通过 `~fmt::basic_memory_buffer` 使用。
  \endrst
 */
  // 模板类 buffer，用于管理数据缓冲区
template <typename T> class buffer {
 private:
  T* ptr_;  // 指向缓冲区数据的指针
  size_t size_;  // 缓冲区中当前元素的数量
  size_t capacity_;  // 缓冲区的容量

 protected:
  // 不初始化 ptr_，因为它不会被访问，以节省一些周期
  FMT_SUPPRESS_MSC_WARNING(26495)
  // 构造函数，初始化 size_ 和 capacity_
  buffer(size_t sz) FMT_NOEXCEPT : size_(sz), capacity_(sz) {}

  // 构造函数，初始化 ptr_、size_ 和 capacity_
  buffer(T* p = nullptr, size_t sz = 0, size_t cap = 0) FMT_NOEXCEPT
      : ptr_(p),
        size_(sz),
        capacity_(cap) {}

  // 默认析构函数
  ~buffer() = default;

  /** 设置缓冲区数据和容量。 */
  void set(T* buf_data, size_t buf_capacity) FMT_NOEXCEPT {
    ptr_ = buf_data;
    capacity_ = buf_capacity;
  }

  /** 增加缓冲区容量，以容纳至少 *capacity* 个元素。 */
  virtual void grow(size_t capacity) = 0;

 public:
  using value_type = T;
  using const_reference = const T&;

  // 禁用拷贝构造函数和赋值运算符重载
  buffer(const buffer&) = delete;
  void operator=(const buffer&) = delete;

  // 返回指向缓冲区数据起始位置的指针
  T* begin() FMT_NOEXCEPT { return ptr_; }
  // 返回指向缓冲区数据末尾位置的指针
  T* end() FMT_NOEXCEPT { return ptr_ + size_; }

  // 返回指向缓冲区数据起始位置的常量指针
  const T* begin() const FMT_NOEXCEPT { return ptr_; }
  // 返回指向缓冲区数据末尾位置的常量指针
  const T* end() const FMT_NOEXCEPT { return ptr_ + size_; }

  /** 返回缓冲区的大小。 */
  size_t size() const FMT_NOEXCEPT { return size_; }

  /** 返回缓冲区的容量。 */
  size_t capacity() const FMT_NOEXCEPT { return capacity_; }

  /** 返回指向缓冲区数据的指针。 */
  T* data() FMT_NOEXCEPT { return ptr_; }

  /** 返回指向缓冲区数据的常量指针。 */
  const T* data() const FMT_NOEXCEPT { return ptr_; }

  /** 清空缓冲区。 */
  void clear() { size_ = 0; }

  // 尝试调整缓冲区大小以容纳 *count* 个元素。如果 T 是 POD 类型，则新元素可能不会被初始化。
  void try_resize(size_t count) {
    try_reserve(count);
    // 如果 count 小于等于 capacity_，则 size_ 等于 count，否则 size_ 等于 capacity_
    size_ = count <= capacity_ ? count : capacity_;
  }

  // 尝试增加缓冲区的容量到 *new_capacity*。它可以以比请求的更小的量增加容量，但保证至少有空间来增加一个额外的元素，要么通过增加容量，要么通过刷新缓冲区（如果已满）。
  void try_reserve(size_t new_capacity) {
    // 如果新的容量大于当前容量，调用 grow 函数增加容量
    if (new_capacity > capacity_) grow(new_capacity);
  }

  // 将 value 添加到缓冲区的末尾
  void push_back(const T& value) {
    // 尝试增加缓冲区的容量到 size_ + 1
    try_reserve(size_ + 1);
    // 将 value 添加到 ptr_ 数组的末尾，并增加 size_
    ptr_[size_++] = value;
  }

  /** 将数据追加到缓冲区的末尾。 */
  template <typename U> void append(const U* begin, const U* end);

  // 重载操作符[]，返回索引为 index 的元素的引用
  template <typename I> T& operator[](I index) { return ptr_[index]; }
  // 重载操作符[]，返回索引为 index 的元素的常量引用
  template <typename I> const T& operator[](I index) const {
    return ptr_[index];
  }
};

// 定义一个结构体 buffer_traits
struct buffer_traits {
  explicit buffer_traits(size_t) {} // 定义一个接受 size_t 参数的构造函数
  size_t count() const { return 0; } // 定义一个返回计数值的函数
  size_t limit(size_t size) { return size; } // 定义一个返回限制值的函数
};

// 定义一个类 fixed_buffer_traits
class fixed_buffer_traits {
 private:
  size_t count_ = 0; // 初始化 count_ 为 0
  size_t limit_; // 定义 limit_ 变量

 public:
  explicit fixed_buffer_traits(size_t limit) : limit_(limit) {} // 定义一个接受 limit 参数的构造函数
  size_t count() const { return count_; } // 定义一个返回计数值的函数
  size_t limit(size_t size) { // 定义一个返回限制值的函数
    size_t n = limit_ - count_; // 计算剩余可写入的空间
    count_ += size; // 更新已写入的数量
    return size < n ? size : n; // 返回可写入的最大数量
  }
};

// 定义一个模板类 iterator_buffer，继承自 Traits 和 buffer<T>
template <typename OutputIt, typename T, typename Traits = buffer_traits>
class iterator_buffer : public Traits, public buffer<T> {
 private:
  OutputIt out_; // 定义一个输出迭代器
  enum { buffer_size = 256 }; // 定义一个枚举类型 buffer_size

 protected:
  void grow(size_t) final FMT_OVERRIDE { // 定义一个虚函数 grow
    if (this->size() == buffer_size) flush(); // 如果缓冲区已满，则刷新
  }
  void flush(); // 声明一个刷新函数

 public:
  explicit iterator_buffer(OutputIt out, size_t n = buffer_size) // 定义一个接受输出迭代器和大小参数的构造函数
      : Traits(n), // 调用 Traits 的构造函数
        buffer<T>(data_, 0, n < size_t(buffer_size) ? n : size_t(buffer_size)), // 初始化 buffer
        out_(out) {} // 初始化输出迭代器
  ~iterator_buffer() { flush(); } // 析构函数，刷新缓冲区

  OutputIt out() { // 定义一个返回输出迭代器的函数
    flush(); // 刷新缓冲区
    return out_; // 返回输出迭代器
  }
  size_t count() const { return Traits::count() + this->size(); } // 返回已写入的数量
};

// 定义一个模板类 iterator_buffer，继承自 buffer<T>
template <typename T> class iterator_buffer<T*, T> : public buffer<T> {
 protected:
  void grow(size_t) final FMT_OVERRIDE {} // 定义一个虚函数 grow

 public:
  explicit iterator_buffer(T* out, size_t = 0) : buffer<T>(out, 0, ~size_t()) {} // 定义一个接受指针和大小参数的构造函数

  T* out() { return &*this->end(); } // 返回写入位置的指针
};

// 定义一个模板类 iterator_buffer，继承自 buffer<typename Container::value_type>
template <typename Container>
class iterator_buffer<std::back_insert_iterator<Container>,
                      enable_if_t<is_contiguous<Container>::value,
                                  typename Container::value_type>>
    : public buffer<typename Container::value_type> {
 private:
  Container& container_; // 定义一个容器的引用

 protected:
  void grow(size_t capacity) final FMT_OVERRIDE { // 定义一个虚函数 grow
    container_.resize(capacity); // 调整容器的大小
    // 调用 set 方法，将容器的第一个元素地址和容量作为参数传入
    this->set(&container_[0], capacity);
  }

 public:
  // 使用容器的大小作为参数，创建迭代器缓冲区
  explicit iterator_buffer(Container& c)
      : buffer<typename Container::value_type>(c.size()), container_(c) {}
  // 使用输出迭代器和大小作为参数，创建迭代器缓冲区
  explicit iterator_buffer(std::back_insert_iterator<Container> out, size_t = 0)
      : iterator_buffer(get_container(out)) {}
  // 返回容器的后插入迭代器
  std::back_insert_iterator<Container> out() {
    return std::back_inserter(container_);
  }
// 结束了一个类的定义

// 一个计数缓冲区，用于计算写入的代码单元数量，丢弃输出
template <typename T = char> class counting_buffer : public buffer<T> {
 private:
  enum { buffer_size = 256 };
  T data_[buffer_size];
  size_t count_ = 0;

 protected:
  // 重写基类的 grow 方法
  void grow(size_t) final FMT_OVERRIDE {
    // 如果大小不等于缓冲区大小，则返回
    if (this->size() != buffer_size) return;
    // 增加计数并清空缓冲区
    count_ += this->size();
    this->clear();
  }

 public:
  // 构造函数，初始化缓冲区
  counting_buffer() : buffer<T>(data_, 0, buffer_size) {}

  // 返回计数
  size_t count() { return count_ + this->size(); }
};

// 一个输出迭代器，用于追加到缓冲区
// 用于减少常见情况下的符号大小
template <typename T>
class buffer_appender : public std::back_insert_iterator<buffer<T>> {
  using base = std::back_insert_iterator<buffer<T>>;

 public:
  // 构造函数，初始化基类
  explicit buffer_appender(buffer<T>& buf) : base(buf) {}
  buffer_appender(base it) : base(it) {}

  // 重载 ++ 操作符
  buffer_appender& operator++() {
    base::operator++();
    return *this;
  }

  buffer_appender operator++(int) {
    buffer_appender tmp = *this;
    ++*this;
    return tmp;
  }
};

// 将输出迭代器映射到缓冲区
template <typename T, typename OutputIt>
iterator_buffer<OutputIt, T> get_buffer(OutputIt);
template <typename T> buffer<T>& get_buffer(buffer_appender<T>);

template <typename OutputIt> OutputIt get_buffer_init(OutputIt out) {
  return out;
}
template <typename T> buffer<T>& get_buffer_init(buffer_appender<T> out) {
  return get_container(out);
}

// 获取迭代器
template <typename Buffer>
auto get_iterator(Buffer& buf) -> decltype(buf.out()) {
  return buf.out();
}
template <typename T> buffer_appender<T> get_iterator(buffer<T>& buf) {
  return buffer_appender<T>(buf);
}

template <typename T, typename Char = char, typename Enable = void>
struct fallback_formatter {
  fallback_formatter() = delete;
};

// 指定 T 是否有启用的 fallback_formatter 特化
template <typename T, typename Context>
using has_fallback_formatter =
    // 检查是否可以构造fallback_formatter对象，使用了模板元编程中的is_constructible类型特性
    std::is_constructible<fallback_formatter<T, typename Context::char_type>>;
// 定义一个空的视图结构体
struct view {};

// 带有名称的参数结构体，继承自视图结构体
template <typename Char, typename T> struct named_arg : view {
  const Char* name;  // 参数名称
  const T& value;     // 参数值的引用
  named_arg(const Char* n, const T& v) : name(n), value(v) {}  // 构造函数，初始化参数名称和值
};

// 带有名称的参数信息结构体
template <typename Char> struct named_arg_info {
  const Char* name;  // 参数名称
  int id;            // 参数ID
};

// 参数数据结构模板，包含参数和带名称的参数
template <typename T, typename Char, size_t NUM_ARGS, size_t NUM_NAMED_ARGS>
struct arg_data {
  // args_[0].named_args points to named_args_ to avoid bloating format_args.
  // +1 to workaround a bug in gcc 7.5 that causes duplicated-branches warning.
  T args_[1 + (NUM_ARGS != 0 ? NUM_ARGS : +1)];  // 参数数组
  named_arg_info<Char> named_args_[NUM_NAMED_ARGS];  // 带名称的参数信息数组

  // 构造函数，初始化参数和带名称的参数
  template <typename... U>
  arg_data(const U&... init) : args_{T(named_args_, NUM_NAMED_ARGS), init...} {}
  // 删除拷贝构造函数
  arg_data(const arg_data& other) = delete;
  // 返回参数数组的指针
  const T* args() const { return args_ + 1; }
  // 返回带名称的参数信息数组的指针
  named_arg_info<Char>* named_args() { return named_args_; }
};

// 参数数据结构模板的特化版本，不包含带名称的参数
template <typename T, typename Char, size_t NUM_ARGS>
struct arg_data<T, Char, NUM_ARGS, 0> {
  // +1 to workaround a bug in gcc 7.5 that causes duplicated-branches warning.
  T args_[NUM_ARGS != 0 ? NUM_ARGS : +1];  // 参数数组

  // 构造函数，初始化参数
  FMT_INLINE arg_data(const U&... init) : args_{init...} {}
  // 返回参数数组的指针
  FMT_INLINE const T* args() const { return args_; }
  // 返回空指针
  FMT_INLINE std::nullptr_t named_args() { return nullptr; }
};

// 初始化带名称的参数信息数组
template <typename Char>
inline void init_named_args(named_arg_info<Char>*, int, int) {}

// 初始化带名称的参数信息数组
template <typename Char, typename T, typename... Tail>
void init_named_args(named_arg_info<Char>* named_args, int arg_count,
                     int named_arg_count, const T&, const Tail&... args) {
  init_named_args(named_args, arg_count + 1, named_arg_count, args...);
}
// 初始化命名参数数组，将命名参数信息存储到数组中
void init_named_args(named_arg_info<Char>* named_args, int arg_count,
                     int named_arg_count, const named_arg<Char, T>& arg,
                     const Tail&... args) {
  // 将命名参数的名称和参数位置存储到数组中
  named_args[named_arg_count++] = {arg.name, arg_count};
  // 递归调用，处理剩余的参数
  init_named_args(named_args, arg_count + 1, named_arg_count, args...);
}

// 当参数为空时，初始化命名参数数组
template <typename... Args>
FMT_INLINE void init_named_args(std::nullptr_t, int, int, const Args&...) {}

// 判断是否为命名参数的类型
template <typename T> struct is_named_arg : std::false_type {};

template <typename T, typename Char>
struct is_named_arg<named_arg<Char, T>> : std::true_type {};

// 计算命名参数的数量
template <bool B = false> constexpr size_t count() { return B ? 1 : 0; }
template <bool B1, bool B2, bool... Tail> constexpr size_t count() {
  return (B1 ? 1 : 0) + count<B2, Tail...>();
}

template <typename... Args> constexpr size_t count_named_args() {
  return count<is_named_arg<Args>::value...>();
}

// 定义枚举类型，表示参数的类型
enum class type {
  none_type,
  int_type,
  uint_type,
  long_long_type,
  ulong_long_type,
  int128_type,
  uint128_type,
  bool_type,
  char_type,
  last_integer_type = char_type,
  float_type,
  double_type,
  long_double_type,
  last_numeric_type = long_double_type,
  cstring_type,
  string_type,
  pointer_type,
  custom_type
};

// 将核心类型映射到相应的类型枚举常量
template <typename T, typename Char>
struct type_constant : std::integral_constant<type, type::custom_type> {};

// 定义宏，用于简化类型常量的定义
#define FMT_TYPE_CONSTANT(Type, constant) \
  template <typename Char>                \
  struct type_constant<Type, Char>        \
      : std::integral_constant<type, type::constant> {}

// 定义各种类型的常量
FMT_TYPE_CONSTANT(int, int_type);
FMT_TYPE_CONSTANT(unsigned, uint_type);
FMT_TYPE_CONSTANT(long long, long_long_type);
FMT_TYPE_CONSTANT(unsigned long long, ulong_long_type);
FMT_TYPE_CONSTANT(int128_t, int128_type);
FMT_TYPE_CONSTANT(uint128_t, uint128_type);
FMT_TYPE_CONSTANT(bool, bool_type);
// 定义宏，用于创建常量类型的格式化参数
FMT_TYPE_CONSTANT(Char, char_type);
FMT_TYPE_CONSTANT(float, float_type);
FMT_TYPE_CONSTANT(double, double_type);
FMT_TYPE_CONSTANT(long double, long_double_type);
FMT_TYPE_CONSTANT(const Char*, cstring_type);
FMT_TYPE_CONSTANT(basic_string_view<Char>, string_type);
FMT_TYPE_CONSTANT(const void*, pointer_type);

// 判断是否为整数类型
constexpr bool is_integral_type(type t) {
  return t > type::none_type && t <= type::last_integer_type;
}

// 判断是否为算术类型
constexpr bool is_arithmetic_type(type t) {
  return t > type::none_type && t <= type::last_numeric_type;
}

// 定义字符串值结构体模板
template <typename Char> struct string_value {
  const Char* data;
  size_t size;
};

// 定义命名参数值结构体模板
template <typename Char> struct named_arg_value {
  const named_arg_info<Char>* data;
  size_t size;
};

// 定义自定义值结构体模板
template <typename Context> struct custom_value {
  using parse_context = typename Context::parse_context_type;
  const void* value;
  void (*format)(const void* arg, parse_context& parse_ctx, Context& ctx);
};

// 格式化参数值类模板
template <typename Context> class value {
 public:
  using char_type = typename Context::char_type;

  union {
    int int_value;
    unsigned uint_value;
    long long long_long_value;
    unsigned long long ulong_long_value;
    int128_t int128_value;
    uint128_t uint128_value;
    bool bool_value;
    char_type char_value;
    float float_value;
    double double_value;
    long double long_double_value;
    const void* pointer;
    string_value<char_type> string;
    custom_value<Context> custom;
    // 定义一个结构体，用于存储命名参数的键值对
    named_arg_value<char_type> named_args;
  };

  // 构造函数，初始化整型值
  constexpr FMT_INLINE value(int val = 0) : int_value(val) {}
  // 构造函数，初始化无符号整型值
  constexpr FMT_INLINE value(unsigned val) : uint_value(val) {}
  // 构造函数，初始化长整型值
  FMT_INLINE value(long long val) : long_long_value(val) {}
  // 构造函数，初始化无符号长整型值
  FMT_INLINE value(unsigned long long val) : ulong_long_value(val) {}
  // 构造函数，初始化128位整型值
  FMT_INLINE value(int128_t val) : int128_value(val) {}
  // 构造函数，初始化无符号128位整型值
  FMT_INLINE value(uint128_t val) : uint128_value(val) {}
  // 构造函数，初始化单精度浮点数值
  FMT_INLINE value(float val) : float_value(val) {}
  // 构造函数，初始化双精度浮点数值
  FMT_INLINE value(double val) : double_value(val) {}
  // 构造函数，初始化长双精度浮点数值
  FMT_INLINE value(long double val) : long_double_value(val) {}
  // 构造函数，初始化布尔值
  FMT_INLINE value(bool val) : bool_value(val) {}
  // 构造函数，初始化字符值
  FMT_INLINE value(char_type val) : char_value(val) {}
  // 构造函数，初始化字符串值
  FMT_INLINE value(const char_type* val) { string.data = val; }
  // 构造函数，初始化字符串视图值
  FMT_INLINE value(basic_string_view<char_type> val) {
    string.data = val.data();
    string.size = val.size();
  }
  // 构造函数，初始化指针值
  FMT_INLINE value(const void* val) : pointer(val) {}
  // 构造函数，初始化命名参数信息
  FMT_INLINE value(const named_arg_info<char_type>* args, size_t size)
      : named_args{args, size} {}

  // 模板构造函数，初始化自定义类型值
  template <typename T> FMT_INLINE value(const T& val) {
    custom.value = &val;
    // 通过上下文获取格式化器类型，允许不同上下文有不同的扩展点
    custom.format = format_custom_arg<
        T, conditional_t<has_formatter<T, Context>::value,
                         typename Context::template formatter_type<T>,
                         fallback_formatter<T, char_type>>>;
  }

 private:
  // 格式化自定义类型参数的私有成员函数
  // 例如用户定义的类
  template <typename T, typename Formatter>
  static void format_custom_arg(const void* arg,
                                typename Context::parse_context_type& parse_ctx,
                                Context& ctx) {
    Formatter f;
    parse_ctx.advance_to(f.parse(parse_ctx));
    ctx.advance_to(f.format(*static_cast<const T*>(arg), ctx));
  }
};

// 创建格式化参数的函数模板，接受上下文和值作为参数
template <typename Context, typename T>
FMT_CONSTEXPR basic_format_arg<Context> make_arg(const T& value);

// 为了最小化需要处理的类型数量，将 long 转换为 int 或 long long，取决于其大小
enum { long_short = sizeof(long) == sizeof(int) };
using long_type = conditional_t<long_short, int, long long>;
using ulong_type = conditional_t<long_short, unsigned, unsigned long long>;

// 表示无法格式化的类型
struct unformattable {};

// 将格式化参数映射到核心类型
template <typename Context> struct arg_mapper {
  using char_type = typename Context::char_type;

  // 将 signed char 映射为 int
  FMT_CONSTEXPR int map(signed char val) { return val; }
  // 将 unsigned char 映射为 unsigned
  FMT_CONSTEXPR unsigned map(unsigned char val) { return val; }
  // 将 short 映射为 int
  FMT_CONSTEXPR int map(short val) { return val; }
  // 将 unsigned short 映射为 unsigned
  FMT_CONSTEXPR unsigned map(unsigned short val) { return val; }
  // 将 int 映射为 int
  FMT_CONSTEXPR int map(int val) { return val; }
  // 将 unsigned 映射为 unsigned
  FMT_CONSTEXPR unsigned map(unsigned val) { return val; }
  // 将 long 映射为 long_type
  FMT_CONSTEXPR long_type map(long val) { return val; }
  // 将 unsigned long 映射为 ulong_type
  FMT_CONSTEXPR ulong_type map(unsigned long val) { return val; }
  // 将 long long 映射为 long long
  FMT_CONSTEXPR long long map(long long val) { return val; }
  // 将 unsigned long long 映射为 unsigned long long
  FMT_CONSTEXPR unsigned long long map(unsigned long long val) { return val; }
  // 将 int128_t 映射为 int128_t
  FMT_CONSTEXPR int128_t map(int128_t val) { return val; }
  // 将 uint128_t 映射为 uint128_t
  FMT_CONSTEXPR uint128_t map(uint128_t val) { return val; }
  // 将 bool 映射为 bool
  FMT_CONSTEXPR bool map(bool val) { return val; }

  // 将字符类型映射为 char_type
  template <typename T, FMT_ENABLE_IF(is_char<T>::value)>
  FMT_CONSTEXPR char_type map(T val) {
    static_assert(
        std::is_same<T, char>::value || std::is_same<T, char_type>::value,
        "mixing character types is disallowed");
  // 返回输入值
  return val;
}

// 映射函数，对于 float 类型的输入，返回输入值
FMT_CONSTEXPR float map(float val) { return val; }
// 映射函数，对于 double 类型的输入，返回输入值
FMT_CONSTEXPR double map(double val) { return val; }
// 映射函数，对于 long double 类型的输入，返回输入值
FMT_CONSTEXPR long double map(long double val) { return val; }

// 映射函数，对于 char_type* 类型的输入，返回输入值
FMT_CONSTEXPR const char_type* map(char_type* val) { return val; }
// 映射函数，对于 const char_type* 类型的输入，返回输入值
FMT_CONSTEXPR const char_type* map(const char_type* val) { return val; }
// 映射函数，对于字符串类型的输入，返回字符串的视图
template <typename T, FMT_ENABLE_IF(is_string<T>::value)>
FMT_CONSTEXPR basic_string_view<char_type> map(const T& val) {
  static_assert(std::is_same<char_type, char_t<T>>::value,
                "mixing character types is disallowed");
  return to_string_view(val);
}
// 映射函数，对于其他类型的输入，返回输入值的视图
template <typename T,
          FMT_ENABLE_IF(
              std::is_constructible<basic_string_view<char_type>, T>::value &&
              !is_string<T>::value && !has_formatter<T, Context>::value &&
              !has_fallback_formatter<T, Context>::value)>
FMT_CONSTEXPR basic_string_view<char_type> map(const T& val) {
  return basic_string_view<char_type>(val);
}
// 映射函数，对于其他类型的输入，返回输入值的视图
template <
    typename T,
    FMT_ENABLE_IF(
        std::is_constructible<std_string_view<char_type>, T>::value &&
        !std::is_constructible<basic_string_view<char_type>, T>::value &&
        !is_string<T>::value && !has_formatter<T, Context>::value &&
        !has_fallback_formatter<T, Context>::value)>
FMT_CONSTEXPR basic_string_view<char_type> map(const T& val) {
  return std_string_view<char_type>(val);
}
// 映射函数，对于 signed char* 类型的输入，返回输入值的转换后的视图
FMT_CONSTEXPR const char* map(const signed char* val) {
  static_assert(std::is_same<char_type, char>::value, "invalid string type");
  return reinterpret_cast<const char*>(val);
}
// 映射函数，对于 unsigned char* 类型的输入，返回输入值的转换后的视图
FMT_CONSTEXPR const char* map(const unsigned char* val) {
  static_assert(std::is_same<char_type, char>::value, "invalid string type");
  return reinterpret_cast<const char*>(val);
}
// 映射函数，对于 signed char* 类型的输入，返回输入值的转换后的视图
FMT_CONSTEXPR const char* map(signed char* val) {
  const auto* const_val = val;
  return map(const_val);
}
// 映射函数，对于 unsigned char* 类型的输入，返回输入值的转换后的视图
FMT_CONSTEXPR const char* map(unsigned char* val) {
    // 将指向常量的指针赋值给 const_val
    const auto* const_val = val;
    // 调用 map 函数并返回结果
    return map(const_val);
  }

  // 将 void* 类型的指针映射为其本身
  FMT_CONSTEXPR const void* map(void* val) { return val; }
  // 将 const void* 类型的指针映射为其本身
  FMT_CONSTEXPR const void* map(const void* val) { return val; }
  // 将 nullptr 映射为其本身
  FMT_CONSTEXPR const void* map(std::nullptr_t val) { return val; }
  // 对于任意类型的指针，禁止格式化，抛出静态断言
  template <typename T> FMT_CONSTEXPR int map(const T*) {
    // 格式化任意指针是不允许的。如果要输出指针，请将其转换为 "void *" 或 "const void *"。
    // 特别地，这禁止了对 "[const] volatile char *" 的格式化，它会被 iostreams 打印为布尔值。
    static_assert(!sizeof(T), "formatting of non-void pointers is disallowed");
    return 0;
  }

  // 对于枚举类型，如果没有自定义格式化器，则将其转换为其基础类型并调用 map 函数
  template <typename T,
            FMT_ENABLE_IF(std::is_enum<T>::value &&
                          !has_formatter<T, Context>::value &&
                          !has_fallback_formatter<T, Context>::value)>
  FMT_CONSTEXPR auto map(const T& val)
      -> decltype(std::declval<arg_mapper>().map(
          static_cast<typename std::underlying_type<T>::type>(val))) {
    return map(static_cast<typename std::underlying_type<T>::type>(val));
  }
  // 对于除字符串和字符类型以外的类型，如果有自定义格式化器，则将其映射为其本身
  template <typename T,
            FMT_ENABLE_IF(!is_string<T>::value && !is_char<T>::value &&
                          (has_formatter<T, Context>::value ||
                           has_fallback_formatter<T, Context>::value)>
  FMT_CONSTEXPR const T& map(const T& val) {
    return val;
  }

  // 对于命名参数，调用 map 函数并返回结果
  template <typename T>
  FMT_CONSTEXPR auto map(const named_arg<char_type, T>& val)
      -> decltype(std::declval<arg_mapper>().map(val.value)) {
    return map(val.value);
  }

  // 对于不可格式化的类型，返回空的 unformattable 对象
  unformattable map(...) { return {}; }
};

// 定义一个模板类型常量，经过 arg_mapper<Context> 映射后的类型
template <typename T, typename Context>
using mapped_type_constant =
    type_constant<decltype(arg_mapper<Context>().map(std::declval<const T&>())),
                  typename Context::char_type>;

// 使用位枚举定义压缩后的参数位数
enum { packed_arg_bits = 4 };
// 最大压缩参数个数
enum { max_packed_args = 62 / packed_arg_bits };
// 无符号长整型，用于表示是否解压缩的位
enum : unsigned long long { is_unpacked_bit = 1ULL << 63 };
// 无符号长整型，用于表示是否有命名参数的位
enum : unsigned long long { has_named_args_bit = 1ULL << 62 };
}  // namespace detail

// 格式化参数，是一个平凡可复制/构造的类型，以便存储在 basic_memory_buffer 中
template <typename Context> class basic_format_arg {
 private:
  detail::value<Context> value_;
  detail::type type_;

  template <typename ContextType, typename T>
  friend FMT_CONSTEXPR basic_format_arg<ContextType> detail::make_arg(
      const T& value);

  template <typename Visitor, typename Ctx>
  friend FMT_CONSTEXPR auto visit_format_arg(Visitor&& vis,
                                             const basic_format_arg<Ctx>& arg)
      -> decltype(vis(0));

  friend class basic_format_args<Context>;
  friend class dynamic_format_arg_store<Context>;

  using char_type = typename Context::char_type;

  template <typename T, typename Char, size_t NUM_ARGS, size_t NUM_NAMED_ARGS>
  friend struct detail::arg_data;

  // 以命名参数信息为参数构造格式化参数
  basic_format_arg(const detail::named_arg_info<char_type>* args, size_t size)
      : value_(args, size) {}

 public:
  // 格式化参数的句柄
  class handle {
   public:
    explicit handle(detail::custom_value<Context> custom) : custom_(custom) {}

    // 格式化参数值
    void format(typename Context::parse_context_type& parse_ctx,
                Context& ctx) const {
      custom_.format(custom_.value, parse_ctx, ctx);
    }

   private:
    detail::custom_value<Context> custom_;
  };

  // 默认构造函数，类型为 none_type
  constexpr basic_format_arg() : type_(detail::type::none_type) {}

  // 显式转换为 bool 类型
  constexpr explicit operator bool() const FMT_NOEXCEPT {
    # 返回类型是否不等于 none_type
    return type_ != detail::type::none_type;
  }

  # 返回类型
  detail::type type() const { return type_; }

  # 返回类型是否为整数类型
  bool is_integral() const { return detail::is_integral_type(type_); }
  # 返回类型是否为算术类型
  bool is_arithmetic() const { return detail::is_arithmetic_type(type_); }
/**
  \rst
  根据参数类型分派到相应的访问方法，例如，如果参数类型是 ``double``，那么将使用类型为 ``double`` 的值调用 ``vis(value)``。
  \endrst
 */
template <typename Visitor, typename Context>
FMT_CONSTEXPR_DECL FMT_INLINE auto visit_format_arg(
    Visitor&& vis, const basic_format_arg<Context>& arg) -> decltype(vis(0)) {
  using char_type = typename Context::char_type;
  switch (arg.type_) {
  case detail::type::none_type:
    break;
  case detail::type::int_type:
    return vis(arg.value_.int_value);
  case detail::type::uint_type:
    return vis(arg.value_.uint_value);
  case detail::type::long_long_type:
    return vis(arg.value_.long_long_value);
  case detail::type::ulong_long_type:
    return vis(arg.value_.ulong_long_value);
#if FMT_USE_INT128
  case detail::type::int128_type:
    return vis(arg.value_.int128_value);
  case detail::type::uint128_type:
    return vis(arg.value_.uint128_value);
#else
  case detail::type::int128_type:
  case detail::type::uint128_type:
    break;
#endif
  case detail::type::bool_type:
    return vis(arg.value_.bool_value);
  case detail::type::char_type:
    return vis(arg.value_.char_value);
  case detail::type::float_type:
    return vis(arg.value_.float_value);
  case detail::type::double_type:
    return vis(arg.value_.double_value);
  case detail::type::long_double_type:
    return vis(arg.value_.long_double_value);
  case detail::type::cstring_type:
    return vis(arg.value_.string.data);
  case detail::type::string_type:
    return vis(basic_string_view<char_type>(arg.value_.string.data,
                                            arg.value_.string.size));
  case detail::type::pointer_type:
    return vis(arg.value_.pointer);
  case detail::type::custom_type:
    return vis(typename basic_format_arg<Context>::handle(arg.value_.custom));
  }
  return vis(monostate());
}
// 定义一个模板结构体，用于判断是否可格式化
template <typename T> struct formattable : std::false_type {};

// 命名空间 detail
namespace detail {

// 一个用于解决在 SFINAE 上下文中使 void_t 工作的 gcc 4.8 的变通方法
template <typename... Ts> struct void_t_impl { using type = void; };

// 使用别名模板 void_t 来定义 void_t_impl 的类型
template <typename... Ts>
using void_t = typename detail::void_t_impl<Ts...>::type;

// 以 SFINAE 友好的方式检测 *任何* 给定类型的迭代器类别
// 不幸的是，旧版本的 std::iterator_traits 不适用于 SFINAE 上下文
template <typename It, typename Enable = void>
struct iterator_category : std::false_type {};

// 针对指针类型的特化，定义其迭代器类别为 std::random_access_iterator_tag
template <typename T> struct iterator_category<T*> {
  using type = std::random_access_iterator_tag;
};

// 针对任意类型的迭代器，检测其是否具有 iterator_category 成员类型
template <typename It>
struct iterator_category<It, void_t<typename It::iterator_category>> {
  using type = typename It::iterator_category;
};

// 检测 *任何* 给定类型是否符合 OutputIterator 概念
template <typename It> class is_output_iterator {
  // 检查可变性，因为所有继承自 std::input_iterator_tag 的迭代器类别
  // *可能* 也满足 OutputIterator 的要求，因此属于 '可变迭代器' 类别
  // [iterator.requirements.general] 第 4 条。编译器只有在 *实际解引用* 迭代器时才会显露这一属性！
  template <typename U>
  static decltype(*(std::declval<U>())) test(std::input_iterator_tag);
  template <typename U> static char& test(std::output_iterator_tag);
  template <typename U> static const char& test(...);

  using type = decltype(test<It>(typename iterator_category<It>::type{}));

 public:
  enum { value = !std::is_const<remove_reference_t<type>>::value };
};

// 检测是否为 back_insert_iterator
template <typename OutputIt>
struct is_back_insert_iterator : std::false_type {};
template <typename Container>
struct is_back_insert_iterator<std::back_insert_iterator<Container>>
    : std::true_type {};

// 检测是否为 contiguous_back_insert_iterator
template <typename OutputIt>
struct is_contiguous_back_insert_iterator : std::false_type {};
// 定义一个模板结构，用于判断是否为连续的后插入迭代器
template <typename Container>
struct is_contiguous_back_insert_iterator<std::back_insert_iterator<Container>>
    : is_contiguous<Container> {};

// 定义一个模板结构，用于判断是否为连续的后插入迭代器，特化为 buffer_appender<Char> 时为 true_type
template <typename Char>
struct is_contiguous_back_insert_iterator<buffer_appender<Char>>
    : std::true_type {};

// 一个类型擦除的 std::locale 的引用，避免引入重量级的 <locale> 头文件
class locale_ref {
 private:
  const void* locale_;  // 一个类型擦除的指向 std::locale 的指针

 public:
  // 默认构造函数，将 locale_ 初始化为 nullptr
  locale_ref() : locale_(nullptr) {}
  // 模板构造函数，接受一个 Locale 类型的参数
  template <typename Locale> explicit locale_ref(const Locale& loc);

  // 转换为 bool 类型，用于判断 locale_ 是否为 nullptr
  explicit operator bool() const FMT_NOEXCEPT { return locale_ != nullptr; }

  // 模板函数，用于获取 Locale 类型的值
  template <typename Locale> Locale get() const;
};

// 一个模板函数，用于编码类型，返回一个无符号长整型
template <typename> constexpr unsigned long long encode_types() { return 0; }

// 一个模板函数，用于编码类型，返回一个无符号长整型
template <typename Context, typename Arg, typename... Args>
constexpr unsigned long long encode_types() {
  return static_cast<unsigned>(mapped_type_constant<Arg, Context>::value) |
         (encode_types<Context, Args...>() << packed_arg_bits);
}

// 一个模板函数，用于创建参数
template <typename Context, typename T>
FMT_CONSTEXPR basic_format_arg<Context> make_arg(const T& value) {
  basic_format_arg<Context> arg;
  arg.type_ = mapped_type_constant<T, Context>::value;
  arg.value_ = arg_mapper<Context>().map(value);
  return arg;
}

// 一个模板函数，用于检查是否为不可格式化类型
template <typename T> int check(unformattable) {
  static_assert(
      formattable<T>(),
      "Cannot format an argument. To make type T formattable provide a "
      "formatter<T> specialization: https://fmt.dev/dev/api.html#udt");
  return 0;
}
// 一个模板函数，用于检查是否为可格式化类型
template <typename T, typename U> inline const U& check(const U& val) {
  return val;
}

// 一个模板函数，用于创建参数，当 IS_PACKED 为 true 时启用 SFINAE
template <bool IS_PACKED, typename Context, type, typename T,
          FMT_ENABLE_IF(IS_PACKED)>
inline value<Context> make_arg(const T& val) {
  return check<T>(arg_mapper<Context>().map(val));
}
// 根据条件是否打包，生成格式化参数
template <bool IS_PACKED, typename Context, type, typename T,
          FMT_ENABLE_IF(!IS_PACKED)>
inline basic_format_arg<Context> make_arg(const T& value) {
  return make_arg<Context>(value);
}

// 判断是否为引用包装器
template <typename T> struct is_reference_wrapper : std::false_type {};
template <typename T>
struct is_reference_wrapper<std::reference_wrapper<T>> : std::true_type {};

// 解开引用包装器
template <typename T> const T& unwrap(const T& v) { return v; }
template <typename T> const T& unwrap(const std::reference_wrapper<T>& v) {
  return static_cast<const T&>(v);
}

// 动态参数列表类
class dynamic_arg_list {
  // 为了解决 clang 的 -Wweak-vtables 警告，使用假模板来避免单一翻译单元的虚函数表问题
  template <typename = void> struct node {
    virtual ~node() = default;
    std::unique_ptr<node<>> next;
  };

  // 有类型的节点
  template <typename T> struct typed_node : node<> {
    T value;

    // 构造函数
    template <typename Arg>
    FMT_CONSTEXPR typed_node(const Arg& arg) : value(arg) {}

    // 构造函数
    template <typename Char>
    FMT_CONSTEXPR typed_node(const basic_string_view<Char>& arg)
        : value(arg.data(), arg.size()) {}
  };

  std::unique_ptr<node<>> head_;

 public:
  // 推入参数
  template <typename T, typename Arg> const T& push(const Arg& arg) {
    auto new_node = std::unique_ptr<typed_node<T>>(new typed_node<T>(arg));
    auto& value = new_node->value;
    new_node->next = std::move(head_);
    head_ = std::move(new_node);
    return value;
  }
};
}  // namespace detail

// 格式化上下文
// 定义一个模板类 basic_format_context，接受两个模板参数 OutputIt 和 Char
template <typename OutputIt, typename Char> class basic_format_context {
 public:
  /** The character type for the output. */
  // 定义输出的字符类型为 Char
  using char_type = Char;

 private:
  // 输出迭代器
  OutputIt out_;
  // 格式化参数
  basic_format_args<basic_format_context> args_;
  // 区域设置引用
  detail::locale_ref loc_;

 public:
  // 输出迭代器类型
  using iterator = OutputIt;
  // 格式化参数类型
  using format_arg = basic_format_arg<basic_format_context>;
  // 解析上下文类型
  using parse_context_type = basic_format_parse_context<Char>;
  // 格式化器类型模板
  template <typename T> using formatter_type = formatter<T, char_type>;

  // 构造函数，初始化输出迭代器、格式化参数和区域设置引用
  basic_format_context(const basic_format_context&) = delete;
  void operator=(const basic_format_context&) = delete;
  /**
   Constructs a ``basic_format_context`` object. References to the arguments are
   stored in the object so make sure they have appropriate lifetimes.
   */
  // 构造函数，初始化输出迭代器、格式化参数和区域设置引用
  basic_format_context(OutputIt out,
                       basic_format_args<basic_format_context> ctx_args,
                       detail::locale_ref loc = detail::locale_ref())
      : out_(out), args_(ctx_args), loc_(loc) {}

  // 获取指定索引的格式化参数
  format_arg arg(int id) const { return args_.get(id); }
  // 获取指定名称的格式化参数
  format_arg arg(basic_string_view<char_type> name) { return args_.get(name); }
  // 获取指定名称的格式化参数的索引
  int arg_id(basic_string_view<char_type> name) { return args_.get_id(name); }
  // 返回格式化参数
  const basic_format_args<basic_format_context>& args() const { return args_; }

  // 错误处理器
  detail::error_handler error_handler() { return {}; }
  // 处理错误信息
  void on_error(const char* message) { error_handler().on_error(message); }

  // 返回输出范围的起始迭代器
  // Returns an iterator to the beginning of the output range.
  iterator out() { return out_; }

  // 将起始迭代器前进到指定位置
  // Advances the begin iterator to ``it``.
  void advance_to(iterator it) {
    if (!detail::is_back_insert_iterator<iterator>()) out_ = it;
  }

  // 返回区域设置引用
  detail::locale_ref locale() { return loc_; }
};

// 使用 buffer_appender 类型作为输出迭代器的 buffer_context 类型
template <typename Char>
using buffer_context =
    basic_format_context<detail::buffer_appender<Char>, Char>;
// 使用 char 类型作为字符类型的 buffer_context 类型
using format_context = buffer_context<char>;
// 使用 wchar_t 类型作为字符类型的 buffer_context 类型
using wformat_context = buffer_context<wchar_t>;
// 定义一个宏，用于解决别名问题
#define FMT_BUFFER_CONTEXT(Char) \
  basic_format_context<detail::buffer_appender<Char>, Char>

/**
  \rst
  对参数的引用数组。可以隐式转换为`~fmt::basic_format_args`，用于传递给类型擦除的格式化函数，比如`~fmt::vformat`。
  \endrst
 */
template <typename Context, typename... Args>
class format_arg_store
#if FMT_GCC_VERSION && FMT_GCC_VERSION < 409
    // 解决 GCC 模板参数替换 bug
    : public basic_format_args<Context>
#endif
{
 private:
  static const size_t num_args = sizeof...(Args); // 参数数量
  static const size_t num_named_args = detail::count_named_args<Args...>(); // 命名参数数量
  static const bool is_packed = num_args <= detail::max_packed_args; // 是否打包

  using value_type = conditional_t<is_packed, detail::value<Context>, basic_format_arg<Context>>; // 值类型

  detail::arg_data<value_type, typename Context::char_type, num_args, num_named_args> data_; // 参数数据

  friend class basic_format_args<Context>; // 声明友元类

  static constexpr unsigned long long desc = // 描述
      (is_packed ? detail::encode_types<Context, Args...>() // 如果是打包，则编码类型
                 : detail::is_unpacked_bit | num_args) | // 否则设置未打包位和参数数量
      (num_named_args != 0 // 如果有命名参数
           ? static_cast<unsigned long long>(detail::has_named_args_bit) // 设置有命名参数位
           : 0);

 public:
  format_arg_store(const Args&... args) // 构造函数
      :
#if FMT_GCC_VERSION && FMT_GCC_VERSION < 409
        basic_format_args<Context>(*this), // 如果是 GCC 版本小于 409，则调用基类构造函数
#endif
        data_{detail::make_arg< // 初始化参数数据
            is_packed, Context,
            detail::mapped_type_constant<Args, Context>::value>(args)...} { // 调用 make_arg 初始化参数
    detail::init_named_args(data_.named_args(), 0, 0, args...); // 初始化命名参数
  }
};
/**
  \rst
  构造一个 `~fmt::format_arg_store` 对象，其中包含对参数的引用，并且可以隐式转换为 `~fmt::format_args`。`Context` 可以省略，默认为 `~fmt::context`。
  有关生存期注意事项，请参见 `~fmt::arg`。
  \endrst
 */
template <typename Context = format_context, typename... Args>
inline format_arg_store<Context, Args...> make_format_args(
    const Args&... args) {
  return {args...};
}

/**
  \rst
  构造一个 `~fmt::format_arg_store` 对象，其中包含对参数的引用，并且可以隐式转换为 `~fmt::format_args`。
  如果 ``format_str`` 是编译时字符串，则 `make_args_checked` 在编译时检查其有效性。
  \endrst
 */
template <typename... Args, typename S, typename Char = char_t<S>>
inline auto make_args_checked(const S& format_str,
                              const remove_reference_t<Args>&... args)
    -> format_arg_store<buffer_context<Char>, remove_reference_t<Args>...> {
  static_assert(
      detail::count<(
              std::is_base_of<detail::view, remove_reference_t<Args>>::value &&
              std::is_reference<Args>::value)...>() == 0,
      "passing views as lvalues is disallowed");
  detail::check_format_string<Args...>(format_str);
  return {args...};
}

/**
  \rst
  返回一个命名参数，用于在格式化函数中使用。它应该只用于调用格式化函数。

  **示例**::

    fmt::print("Elapsed time: {s:.2f} seconds", fmt::arg("s", 1.23));
  \endrst
 */
template <typename Char, typename T>
inline detail::named_arg<Char, T> arg(const Char* name, const T& arg) {
  static_assert(!detail::is_named_arg<T>(), "nested named arguments");
  return {name, arg};
}
/**
  \rst
  A dynamic version of `fmt::format_arg_store`.
  It's equipped with a storage to potentially temporary objects which lifetimes
  could be shorter than the format arguments object.

  It can be implicitly converted into `~fmt::basic_format_args` for passing
  into type-erased formatting functions such as `~fmt::vformat`.
  \endrst
 */
// 定义一个动态版本的 `fmt::format_arg_store` 类
template <typename Context>
class dynamic_format_arg_store
#if FMT_GCC_VERSION && FMT_GCC_VERSION < 409
    // Workaround a GCC template argument substitution bug.
    : public basic_format_args<Context>
#endif
{
 private:
  using char_type = typename Context::char_type;

  // 定义一个模板结构体，用于判断是否需要复制存储
  template <typename T> struct need_copy {
    static constexpr detail::type mapped_type =
        detail::mapped_type_constant<T, Context>::value;

    enum {
      value = !(detail::is_reference_wrapper<T>::value ||
                std::is_same<T, basic_string_view<char_type>>::value ||
                std::is_same<T, detail::std_string_view<char_type>>::value ||
                (mapped_type != detail::type::cstring_type &&
                 mapped_type != detail::type::string_type &&
                 mapped_type != detail::type::custom_type))
    };
  };

  // 定义一个模板别名，用于存储类型
  template <typename T>
  using stored_type = conditional_t<detail::is_string<T>::value,
                                    std::basic_string<char_type>, T>;

  // 存储 basic_format_arg 必须是连续的
  std::vector<basic_format_arg<Context>> data_;
  // 存储命名参数信息
  std::vector<detail::named_arg_info<char_type>> named_info_;

  // 存储不适合放入 basic_format_arg 的参数，必须在不重定位的情况下增长，因为 data_ 中的项引用它
  detail::dynamic_arg_list dynamic_args_;

  friend class basic_format_args<Context>;

  // 获取类型信息
  unsigned long long get_types() const {
  return detail::is_unpacked_bit | data_.size() |
         (named_info_.empty()
              ? 0ULL
              : static_cast<unsigned long long>(detail::has_named_args_bit));
}

const basic_format_arg<Context>* data() const {
  // 如果没有命名参数，返回数据的指针；否则返回数据的指针加一
  return named_info_.empty() ? data_.data() : data_.data() + 1;
}

template <typename T> void emplace_arg(const T& arg) {
  // 将参数添加到数据中
  data_.emplace_back(detail::make_arg<Context>(arg));
}

template <typename T>
void emplace_arg(const detail::named_arg<char_type, T>& arg) {
  if (named_info_.empty()) {
    // 如果没有命名参数，插入一个空指针
    constexpr const detail::named_arg_info<char_type>* zero_ptr{nullptr};
    data_.insert(data_.begin(), {zero_ptr, 0});
  }
  // 将参数添加到数据中
  data_.emplace_back(detail::make_arg<Context>(detail::unwrap(arg.value)));
  auto pop_one = [](std::vector<basic_format_arg<Context>>* data) {
    data->pop_back();
  };
  // 创建一个 unique_ptr 用于管理数据，确保在函数结束时释放资源
  std::unique_ptr<std::vector<basic_format_arg<Context>>, decltype(pop_one)>
      guard{&data_, pop_one};
  // 将命名参数信息添加到 named_info_ 中
  named_info_.push_back({arg.name, static_cast<int>(data_.size() - 2u)});
  // 更新数据的命名参数信息
  data_[0].value_.named_args = {named_info_.data(), named_info_.size()};
  guard.release();
}

public:
/**
  \rst
  将参数添加到动态存储中，以便稍后传递给格式化函数。

  注意，自定义类型和字符串类型（但不包括字符串视图）会被复制到动态存储中，必要时动态分配内存。

  **示例**::

    fmt::dynamic_format_arg_store<fmt::format_context> store;
    store.push_back(42);
    store.push_back("abc");
    store.push_back(1.5f);
    std::string result = fmt::vformat("{} and {} and {}", store);
\endrst
*/
template <typename T> void push_back(const T& arg) {
  // 如果需要复制参数，则调用 emplace_arg 将参数添加到数据中；否则直接添加参数的引用
  if (detail::const_check(need_copy<T>::value))
    emplace_arg(dynamic_args_.push<stored_type<T>>(arg));
  else
    emplace_arg(detail::unwrap(arg));
}

/**
  \rst
  将参数的引用添加到动态存储中，以便稍后传递给
  /**
    一个格式化函数。支持通过 ``std::ref()``/``std::cref()`` 包装的命名参数。

    **示例**::

      fmt::dynamic_format_arg_store<fmt::format_context> store;
      char str[] = "1234567890";
      store.push_back(std::cref(str));
      int a1_val{42};
      auto a1 = fmt::arg("a1_", a1_val);
      store.push_back(std::cref(a1));

      // 更改 str 会影响输出，但仅对字符串和自定义类型有效。
      str[0] = 'X';

      std::string result = fmt::vformat("{} and {a1_}");
      assert(result == "X234567890 and 42");
    \endrst
  */
  template <typename T> void push_back(std::reference_wrapper<T> arg) {
    static_assert(
        detail::is_named_arg<typename std::remove_cv<T>::type>::value ||
            need_copy<T>::value,
        "内置类型和字符串视图的对象始终会被复制");
    emplace_arg(arg.get());
  }

  /**
    将命名参数添加到动态存储中，以便稍后传递给格式化函数。支持 ``std::reference_wrapper`` 以避免复制参数。
  */
  template <typename T>
  void push_back(const detail::named_arg<char_type, T>& arg) {
    const char_type* arg_name =
        dynamic_args_.push<std::basic_string<char_type>>(arg.name).c_str();
    if (detail::const_check(need_copy<T>::value)) {
      emplace_arg(
          fmt::arg(arg_name, dynamic_args_.push<stored_type<T>>(arg.value)));
    } else {
      emplace_arg(fmt::arg(arg_name, arg.value));
    }
  }

  /** 从存储中清除所有元素 */
  void clear() {
    data_.clear();
    named_info_.clear();
    dynamic_args_ = detail::dynamic_arg_list();
  }

  /**
    \rst
    预留空间，以存储至少 *new_cap* 个参数，包括 *new_cap_named* 个命名参数。
    \endrst
  */
  void reserve(size_t new_cap, size_t new_cap_named) {
    FMT_ASSERT(new_cap >= new_cap_named,
               "参数集包括命名参数集");
    data_.reserve(new_cap);
  }
    # 为 named_info_ 预留新的容量
    named_info_.reserve(new_cap_named);
  }
};

/**
  \rst
  A view of a collection of formatting arguments. To avoid lifetime issues it
  should only be used as a parameter type in type-erased functions such as
  ``vformat``::

    void vlog(string_view format_str, format_args args);  // OK
    format_args args = make_format_args(42);  // Error: dangling reference
  \endrst
 */
// 定义一个格式化参数的集合视图
template <typename Context> class basic_format_args {
 public:
  using size_type = int;
  using format_arg = basic_format_arg<Context>;

 private:
  // 包含有关格式化参数的信息的描述符
  // 如果参数数量小于或等于 max_packed_args，则参数类型将在描述符中传递。
  // 这样可以减少每次格式化函数调用的二进制代码大小。
  unsigned long long desc_;
  union {
    // 如果 is_packed() 返回 true，则参数值存储在 values_ 中；
    // 否则它们存储在 args_ 中。这样做是为了提高缓存局部性和减少编译代码大小，
    // 因为存储较大的对象可能需要更多的代码（至少在 x86-64 上是这样），
    // 即使实际上复制到堆栈的数据量相同。这在膨胀测试中节省了约 10%。
    const detail::value<Context>* values_;
    const format_arg* args_;
  };

  // 检查参数是否被打包
  bool is_packed() const { return (desc_ & detail::is_unpacked_bit) == 0; }
  // 检查是否有命名参数
  bool has_named_args() const {
    return (desc_ & detail::has_named_args_bit) != 0;
  }

  // 返回参数的类型
  detail::type type(int index) const {
    int shift = index * detail::packed_arg_bits;
    unsigned int mask = (1 << detail::packed_arg_bits) - 1;
    // 返回通过位运算得到的指定类型的参数值
    return static_cast<detail::type>((desc_ >> shift) & mask);
  }

  // 用于从值数组构造 basic_format_args 对象
  basic_format_args(unsigned long long desc,
                    const detail::value<Context>* values)
      : desc_(desc), values_(values) {}
  // 用于从参数数组构造 basic_format_args 对象
  basic_format_args(unsigned long long desc, const format_arg* args)
      : desc_(desc), args_(args) {}

 public:
  // 默认构造函数
  basic_format_args() : desc_(0) {}

  /**
   \rst
   从 format_arg_store 构造 basic_format_args 对象。
   \endrst
   */
  template <typename... Args>
  FMT_INLINE basic_format_args(const format_arg_store<Context, Args...>& store)
      : basic_format_args(store.desc, store.data_.args()) {}

  /**
   \rst
   从 dynamic_format_arg_store 构造 basic_format_args 对象。
   \endrst
   */
  FMT_INLINE basic_format_args(const dynamic_format_arg_store<Context>& store)
      : basic_format_args(store.get_types(), store.data()) {}

  /**
   \rst
   从动态参数集构造 basic_format_args 对象。
   \endrst
   */
  basic_format_args(const format_arg* args, int count)
      : basic_format_args(detail::is_unpacked_bit | detail::to_unsigned(count),
                          args) {}

  /** 返回指定 id 的参数。 */
  format_arg get(int id) const {
    format_arg arg;
    if (!is_packed()) {
      if (id < max_size()) arg = args_[id];
      return arg;
    }
    if (id >= detail::max_packed_args) return arg;
    arg.type_ = type(id);
    if (arg.type_ == detail::type::none_type) return arg;
    arg.value_ = values_[id];
    return arg;
  }

  template <typename Char> format_arg get(basic_string_view<Char> name) const {
    int id = get_id(name);
    return id >= 0 ? get(id) : format_arg();
  }

  template <typename Char> int get_id(basic_string_view<Char> name) const {
    if (!has_named_args()) return -1;
    const auto& named_args =
        (is_packed() ? values_[-1] : args_[-1].value_).named_args;
    // 遍历 named_args 数组，查找指定 name 对应的 id，并返回
    for (size_t i = 0; i < named_args.size; ++i) {
      if (named_args.data[i].name == name) return named_args.data[i].id;
    }
    // 如果未找到对应的 name，则返回 -1
    return -1;
  }

  // 返回最大大小
  int max_size() const {
    // 获取最大打包参数值
    unsigned long long max_packed = detail::max_packed_args;
    // 如果是打包状态，则返回最大打包参数值，否则返回 desc_ 与 is_unpacked_bit 的按位取反结果
    return static_cast<int>(is_packed() ? max_packed
                                        : desc_ & ~detail::is_unpacked_bit);
  }
};

/** An alias to ``basic_format_args<context>``. */
// 它是一个独立的类型，而不是一个别名，以使符号可读。
struct format_args : basic_format_args<format_context> {
  template <typename... Args>
  FMT_INLINE format_args(const Args&... args) : basic_format_args(args...) {}
};
struct wformat_args : basic_format_args<wformat_context> {
  using basic_format_args::basic_format_args;
};

namespace detail {

template <typename Char, FMT_ENABLE_IF(!std::is_same<Char, char>::value)>
std::basic_string<Char> vformat(
    basic_string_view<Char> format_str,
    basic_format_args<buffer_context<type_identity_t<Char>>> args);

FMT_API std::string vformat(string_view format_str, format_args args);

template <typename Char>
buffer_appender<Char> vformat_to(
    buffer<Char>& buf, basic_string_view<Char> format_str,
    basic_format_args<FMT_BUFFER_CONTEXT(type_identity_t<Char>)> args);

template <typename Char, typename Args,
          FMT_ENABLE_IF(!std::is_same<Char, char>::value)>
inline void vprint_mojibake(std::FILE*, basic_string_view<Char>, const Args&) {}

FMT_API void vprint_mojibake(std::FILE*, string_view, format_args);
#ifndef _WIN32
inline void vprint_mojibake(std::FILE*, string_view, format_args) {}
#endif
}  // namespace detail

/** Formats a string and writes the output to ``out``. */
// GCC 8 and earlier cannot handle std::back_insert_iterator<Container> with
// vformat_to<ArgFormatter>(...) overload, so SFINAE on iterator type instead.
template <typename OutputIt, typename S, typename Char = char_t<S>,
          FMT_ENABLE_IF(detail::is_output_iterator<OutputIt>::value)>
OutputIt vformat_to(
    OutputIt out, const S& format_str,
    basic_format_args<buffer_context<type_identity_t<Char>>> args) {
  decltype(detail::get_buffer<Char>(out)) buf(detail::get_buffer_init(out));
  detail::vformat_to(buf, to_string_view(format_str), args);
  return detail::get_iterator(buf);
}
/**
 \rst
 Formats arguments, writes the result to the output iterator ``out`` and returns
 the iterator past the end of the output range.

 **Example**::

   std::vector<char> out;
   fmt::format_to(std::back_inserter(out), "{}", 42);
 \endrst
 */
// 格式化参数，将结果写入输出迭代器“out”，并返回超出输出范围末尾的迭代器。
template <typename OutputIt, typename S, typename... Args,
          FMT_ENABLE_IF(detail::is_output_iterator<OutputIt>::value&&
                            detail::is_string<S>::value)>
inline OutputIt format_to(OutputIt out, const S& format_str, Args&&... args) {
  const auto& vargs = fmt::make_args_checked<Args...>(format_str, args...);
  return vformat_to(out, to_string_view(format_str), vargs);
}

template <typename OutputIt> struct format_to_n_result {
  /** Iterator past the end of the output range. */
  OutputIt out;
  /** Total (not truncated) output size. */
  size_t size;
};

template <typename OutputIt, typename Char, typename... Args,
          FMT_ENABLE_IF(detail::is_output_iterator<OutputIt>::value)>
inline format_to_n_result<OutputIt> vformat_to_n(
    OutputIt out, size_t n, basic_string_view<Char> format_str,
    basic_format_args<buffer_context<type_identity_t<Char>>> args) {
  detail::iterator_buffer<OutputIt, Char, detail::fixed_buffer_traits> buf(out,
                                                                           n);
  detail::vformat_to(buf, format_str, args);
  return {buf.out(), buf.count()};
}

/**
 \rst
 Formats arguments, writes up to ``n`` characters of the result to the output
 iterator ``out`` and returns the total output size and the iterator past the
 end of the output range.
 \endrst
 */
// 格式化参数，将结果写入输出迭代器“out”，并返回总输出大小和超出输出范围末尾的迭代器。
template <typename OutputIt, typename S, typename... Args,
          FMT_ENABLE_IF(detail::is_string<S>::value&&
                            detail::is_output_iterator<OutputIt>::value)>
// 定义一个模板函数，用于将格式化后的内容写入到输出迭代器中，最多写入 n 个字符
template <typename OutputIt>
inline format_to_n_result<OutputIt> format_to_n(OutputIt out, size_t n,
                                                const S& format_str,
                                                const Args&... args) {
  // 创建参数包
  const auto& vargs = fmt::make_args_checked<Args...>(format_str, args...);
  // 调用 vformat_to_n 函数将格式化后的内容写入到输出迭代器中
  return vformat_to_n(out, n, to_string_view(format_str), vargs);
}

/**
  返回 ``format(format_str, args...)`` 输出的字符数。
 */
template <typename... Args>
inline size_t formatted_size(string_view format_str, Args&&... args) {
  // 创建参数包
  const auto& vargs = fmt::make_args_checked<Args...>(format_str, args...);
  // 创建一个计数缓冲区
  detail::counting_buffer<> buf;
  // 调用 vformat_to 函数将格式化后的内容写入到计数缓冲区中
  detail::vformat_to(buf, format_str, vargs);
  // 返回计数缓冲区中的字符数
  return buf.count();
}

// 根据格式字符串和参数包进行格式化，并返回结果作为字符串
template <typename S, typename Char = char_t<S>>
FMT_INLINE std::basic_string<Char> vformat(
    const S& format_str,
    basic_format_args<buffer_context<type_identity_t<Char>>> args) {
  return detail::vformat(to_string_view(format_str), args);
}

/**
  格式化参数并将结果作为字符串返回。

  **示例**::

    #include <fmt/core.h>
    std::string message = fmt::format("The answer is {}", 42);
*/
// 将 char_t 作为默认模板参数传递，而不是使用 std::basic_string<char_t<S>>，以减少符号大小
template <typename S, typename... Args, typename Char = char_t<S>>
FMT_INLINE std::basic_string<Char> format(const S& format_str, Args&&... args) {
  // 创建参数包
  const auto& vargs = fmt::make_args_checked<Args...>(format_str, args...);
  // 调用 vformat 函数进行格式化，并返回结果作为字符串
  return detail::vformat(to_string_view(format_str), vargs);
}

// 声明 vprint 函数，用于将格式化后的内容写入到字符串视图中
FMT_API void vprint(string_view, format_args);
// 声明 vprint 函数，用于将格式化后的内容写入到文件流中
FMT_API void vprint(std::FILE*, string_view, format_args);

/**
  根据 ``format_str`` 中的规范格式化 ``args``，并将输出写入文件 ``f`` 中。
  假定字符串是 Unicode 编码，除非将宏 ``FMT_UNICODE`` 设置为 0。

  **示例**::

    fmt::print(stderr, "Don't {}!", "panic");
 */
// 使用模板实现的打印函数，可以将格式化后的输出写入到文件流中
template <typename S, typename... Args, typename Char = char_t<S>>
inline void print(std::FILE* f, const S& format_str, Args&&... args) {
  // 创建参数包，用于格式化字符串
  const auto& vargs = fmt::make_args_checked<Args...>(format_str, args...);
  // 如果是 Unicode 编码，则调用 vprint 函数输出
  // 否则调用 vprint_mojibake 函数输出
  return detail::is_unicode<Char>()
             ? vprint(f, to_string_view(format_str), vargs)
             : detail::vprint_mojibake(f, to_string_view(format_str), vargs);
}

/**
  \rst
  根据 ``format_str`` 中的规范格式化 ``args``，并将输出写入到 ``stdout``。
  字符串假定为 Unicode 编码，除非 ``FMT_UNICODE`` 宏设置为 0。

  **Example**::

    fmt::print("Elapsed time: {0:.2f} seconds", 1.23);
  \endrst
 */
// 使用模板实现的打印函数，将格式化后的输出写入到标准输出流中
template <typename S, typename... Args, typename Char = char_t<S>>
inline void print(const S& format_str, Args&&... args) {
  // 创建参数包，用于格式化字符串
  const auto& vargs = fmt::make_args_checked<Args...>(format_str, args...);
  // 如果是 Unicode 编码，则调用 vprint 函数输出
  // 否则调用 vprint_mojibake 函数输出
  return detail::is_unicode<Char>()
             ? vprint(to_string_view(format_str), vargs)
             : detail::vprint_mojibake(stdout, to_string_view(format_str), vargs);
}
// FMT_END_NAMESPACE 宏的结束标记
FMT_END_NAMESPACE

// 结束 FMT_CORE_H_ 宏的定义
#endif  // FMT_CORE_H_
```