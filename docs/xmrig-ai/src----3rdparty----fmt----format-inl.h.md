# `xmrig\src\3rdparty\fmt\format-inl.h`

```cpp
// 格式化库的 C++ 实现
//
// 版权所有 (c) 2012 - 2016, Victor Zverovich
// 保留所有权利
//
// 有关许可信息，请参阅 format.h。

#ifndef FMT_FORMAT_INL_H_
#define FMT_FORMAT_INL_H_

#include <cassert>
#include <cctype>
#include <climits>
#include <cmath>
#include <cstdarg>
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

// 如果相应的系统函数不可用，则调用 strerror_r 和 strerror_s 的虚拟实现。
inline fmt::detail::null<> strerror_r(int, char*, ...) { return {}; }
inline fmt::detail::null<> strerror_s(char*, size_t, ...) { return {}; }

FMT_BEGIN_NAMESPACE
namespace detail {

FMT_FUNC void assert_fail(const char* file, int line, const char* message) {
  // 使用未经检查的 std::fprintf 来避免在写入 stderr 失败时触发另一个断言
  std::fprintf(stderr, "%s:%d: assertion failed: %s", file, line, message);
  // 选择 std::terminate 而不是 std::abort，以满足在设备代码传递期间的 CUDA 模式下的 Clang。
  std::terminate();
}

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

// strerror 的可移植线程安全版本。
// 将 buffer 设置为指向描述错误代码的字符串。
// 这可以是指向存储在 buffer 中的字符串的指针，
// 也可以是指向某些静态不可变字符串的指针。
// 返回以下值之一：
//   0      - 成功
//   ERANGE - 缓冲区不足以存储错误消息
//   其他   - 失败
// 缓冲区至少应为大小为 1。
// 定义一个内联函数，用于安全地获取错误信息并存储在指定的缓冲区中
inline int safe_strerror(int error_code, char*& buffer,
                         size_t buffer_size) FMT_NOEXCEPT {
  // 断言缓冲区不为空且缓冲区大小不为0，否则抛出异常
  FMT_ASSERT(buffer != nullptr && buffer_size != 0, "invalid buffer");

  // 定义一个内部类 dispatcher
  class dispatcher {
   private:
    int error_code_;
    char*& buffer_;
    size_t buffer_size_;

    // 一个空操作的赋值运算符，用于避免错误警告
    void operator=(const dispatcher&) {}

    // 处理 XSI-compliant 版本的 strerror_r 的结果
    int handle(int result) {
      // glibc 版本在2.13之前将结果存储在 errno 中
      return result == -1 ? errno : result;
    }

    // 处理 GNU-specific 版本的 strerror_r 的结果
    FMT_MAYBE_UNUSED
    int handle(char* message) {
      // 如果缓冲区已满，则消息可能被截断
      if (message == buffer_ && strlen(buffer_) == buffer_size_ - 1)
        return ERANGE;
      buffer_ = message;
      return 0;
    }

    // 处理 strerror_r 不可用的情况
    FMT_MAYBE_UNUSED
    int handle(detail::null<>) {
      return fallback(strerror_s(buffer_, buffer_size_, error_code_));
    }

    // 当 strerror_r 不可用时，回退到 strerror_s
    FMT_MAYBE_UNUSED
    int fallback(int result) {
      // 如果缓冲区已满，则消息可能被截断
      return result == 0 && strlen(buffer_) == buffer_size_ - 1 ? ERANGE
                                                                : result;
    }

#if !FMT_MSC_VER
    // 当 strerror_r 和 strerror_s 都不可用时，回退到 strerror
    int fallback(detail::null<>) {
      errno = 0;
      buffer_ = strerror(error_code_);
      return errno;
    }
#endif

   public:
    dispatcher(int err_code, char*& buf, size_t buf_size)
        : error_code_(err_code), buffer_(buf), buffer_size_(buf_size) {}

    // 执行获取错误信息的操作
    int run() { return handle(strerror_r(error_code_, buffer_, buffer_size_)); }
  };
  // 创建 dispatcher 对象并执行获取错误信息的操作
  return dispatcher(error_code, buffer, buffer_size).run();
}
// 格式化错误代码并将结果存储在输出缓冲区中
FMT_FUNC void format_error_code(detail::buffer<char>& out, int error_code,
                                string_view message) FMT_NOEXCEPT {
  // 尝试将输出缓冲区大小调整为0，以避免动态内存分配和潜在的bad_alloc
  out.try_resize(0);
  static const char SEP[] = ": ";
  static const char ERROR_STR[] = "error ";
  // 减去2是为了考虑到SEP和ERROR_STR中的终止空字符
  size_t error_code_size = sizeof(SEP) + sizeof(ERROR_STR) - 2;
  auto abs_value = static_cast<uint32_or_64_or_128_t<int>>(error_code);
  if (detail::is_negative(error_code)) {
    abs_value = 0 - abs_value;
    ++error_code_size;
  }
  error_code_size += detail::to_unsigned(detail::count_digits(abs_value));
  auto it = buffer_appender<char>(out);
  // 如果消息大小小于等于inline_buffer_size减去error_code_size，则将消息和SEP格式化到输出缓冲区中
  if (message.size() <= inline_buffer_size - error_code_size)
    format_to(it, "{}{}", message, SEP);
  // 将ERROR_STR和error_code格式化到输出缓冲区中
  format_to(it, "{}{}", ERROR_STR, error_code);
  // 断言输出缓冲区大小不超过inline_buffer_size
  assert(out.size() <= inline_buffer_size);
}

// 报告错误，将格式化后的错误消息输出到stderr
FMT_FUNC void report_error(format_func func, int error_code,
                           string_view message) FMT_NOEXCEPT {
  // 创建一个内存缓冲区来存储完整的错误消息
  memory_buffer full_message;
  // 调用func函数将错误代码和消息格式化到full_message中
  func(full_message, error_code, message);
  // 不使用fwrite_fully，因为后者可能会抛出异常
  (void)std::fwrite(full_message.data(), full_message.size(), 1, stderr);
  // 输出换行符到stderr
  std::fputc('\n', stderr);
}

// 一个包装在fwrite周围的函数，当出现错误时会抛出异常
inline void fwrite_fully(const void* ptr, size_t size, size_t count,
                         FILE* stream) {
  // 调用标准库的fwrite函数将数据写入流中
  size_t written = std::fwrite(ptr, size, count, stream);
  // 如果写入的数量小于count，则抛出system_error异常
  if (written < count) FMT_THROW(system_error(errno, "cannot write to file"));
}
}  // namespace detail

// 如果FMT_STATIC_THOUSANDS_SEPARATOR未定义，则定义detail命名空间
#if !defined(FMT_STATIC_THOUSANDS_SEPARATOR)
namespace detail {

// locale_ref类的构造函数，接受一个Locale类型的参数
template <typename Locale>
locale_ref::locale_ref(const Locale& loc) : locale_(&loc) {
  // 静态断言，确保Locale类型是std::locale
  static_assert(std::is_same<Locale, std::locale>::value, "");
}
template <typename Locale> Locale locale_ref::get() const {
  // 确保模板参数 Locale 是 std::locale 类型
  static_assert(std::is_same<Locale, std::locale>::value, "");
  // 如果 locale_ 不为空，则返回其指向的 std::locale 对象，否则返回默认构造的 std::locale 对象
  return locale_ ? *static_cast<const std::locale*>(locale_) : std::locale();
}

template <typename Char> FMT_FUNC std::string grouping_impl(locale_ref loc) {
  // 使用 std::use_facet 获取 loc 中的 std::numpunct<Char> facet，并调用其 grouping() 方法
  return std::use_facet<std::numpunct<Char>>(loc.get<std::locale>()).grouping();
}
template <typename Char> FMT_FUNC Char thousands_sep_impl(locale_ref loc) {
  // 使用 std::use_facet 获取 loc 中的 std::numpunct<Char> facet，并调用其 thousands_sep() 方法
  return std::use_facet<std::numpunct<Char>>(loc.get<std::locale>())
      .thousands_sep();
}
template <typename Char> FMT_FUNC Char decimal_point_impl(locale_ref loc) {
  // 使用 std::use_facet 获取 loc 中的 std::numpunct<Char> facet，并调用其 decimal_point() 方法
  return std::use_facet<std::numpunct<Char>>(loc.get<std::locale>())
      .decimal_point();
}
}  // namespace detail
#else
template <typename Char>
FMT_FUNC std::string detail::grouping_impl(locale_ref) {
  // 返回默认的分组规则 "\03"
  return "\03";
}
template <typename Char> FMT_FUNC Char detail::thousands_sep_impl(locale_ref) {
  // 返回静态千位分隔符 FMT_STATIC_THOUSANDS_SEPARATOR
  return FMT_STATIC_THOUSANDS_SEPARATOR;
}
template <typename Char> FMT_FUNC Char detail::decimal_point_impl(locale_ref) {
  // 返回默认的小数点符号 '.'
  return '.';
}
#endif

FMT_API FMT_FUNC format_error::~format_error() FMT_NOEXCEPT = default;
FMT_API FMT_FUNC system_error::~system_error() FMT_NOEXCEPT = default;

FMT_FUNC void system_error::init(int err_code, string_view format_str,
                                 format_args args) {
  // 初始化 error_code_ 为 err_code
  error_code_ = err_code;
  // 创建内存缓冲区 buffer
  memory_buffer buffer;
  // 格式化系统错误信息并将结果存入 buffer
  format_system_error(buffer, err_code, vformat(format_str, args));
  // 将 buffer 转换为 std::runtime_error 对象，并赋值给 base
  std::runtime_error& base = *this;
  base = std::runtime_error(to_string(buffer));
}

namespace detail {

template <> FMT_FUNC int count_digits<4>(detail::fallback_uintptr n) {
  // fallback_uintptr 总是以小端存储
  int i = static_cast<int>(sizeof(void*)) - 1;
  // 从高位开始遍历 n.value，找到第一个非零的字节
  while (i > 0 && n.value[i] == 0) --i;
  // 计算每个字节中非零位的数量，并返回总位数
  auto char_digits = std::numeric_limits<unsigned char>::digits / 4;
  return i >= 0 ? i * char_digits + count_digits<4, unsigned>(n.value[i]) : 1;
}

template <typename T>
// 定义一个静态成员变量，存储数字字符对应的数字值
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

// 定义一个静态成员变量，存储16进制字符
template <typename T>
const char basic_data<T>::hex_digits[] = "0123456789abcdef";

// 定义一个宏，用于生成10的幂的倍数
#define FMT_POWERS_OF_10(factor)                                             \
  factor * 10, (factor)*100, (factor)*1000, (factor)*10000, (factor)*100000, \
      (factor)*1000000, (factor)*10000000, (factor)*100000000,               \
      (factor)*1000000000

// 定义一个静态成员变量，存储64位整数的10的幂
template <typename T>
const uint64_t basic_data<T>::powers_of_10_64[] = {
    1, FMT_POWERS_OF_10(1), FMT_POWERS_OF_10(1000000000ULL),
    10000000000000000000ULL};

// 定义一个模板类的静态成员变量
template <typename T>
// 32位整数类型的零或10的幂的数组
const uint32_t basic_data<T>::zero_or_powers_of_10_32[] = {0, 0, FMT_POWERS_OF_10(1)};

// 模板函数，64位整数类型的零或10的幂的数组
template <typename T>
const uint64_t basic_data<T>::zero_or_powers_of_10_64[] = {
    0, 0, FMT_POWERS_OF_10(1), FMT_POWERS_OF_10(1000000000ULL),
    10000000000000000000ULL};

// 64位整数类型的 pow(10, k) 的标准化尾数，其中 k = -348, -340, ..., 340
// 这些数据是由 support/compute-powers.py 生成的
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
    # 这些是十六进制数值，可能是用作密钥或哈希值
    0xf6c69a72a3989f5c, 0xb7dcbf5354e9bece, 0x88fcf317f22241e2,
    0xcc20ce9bd35c78a5, 0x98165af37b2153df, 0xe2a0b5dc971f303a,
    0xa8d9d1535ce3b396, 0xfb9b7cd9a4a7443c, 0xbb764c4ca7a44410,
    0x8bab8eefb6409c1a, 0xd01fef10a657842c, 0x9b10a4e5e9913129,
    0xe7109bfba19c0c9d, 0xac2820d9623bf429, 0x80444b5e7aa7cf85,
    0xbf21e44003acdd2d, 0x8e679c2f5e44ff8f, 0xd433179d9c8cb841,
    0x9e19db92b4e31ba9, 0xeb96bf6ebadf77d9, 0xaf87023b9bf0ee6b,
};

// 用于存储 pow(10, k) 的二进制指数，其中 k = -348, -340, ..., 340，对应于上面的有效数字。
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

// 存储用于 pow(5, k) 的除法测试表的条目，其中 k = 32 位无符号整数
template <typename T>
const divtest_table_entry<uint32_t> basic_data<T>::divtest_table_for_pow5_32[] =
    {{0x00000001, 0xffffffff}, {0xcccccccd, 0x33333333},
     {0xc28f5c29, 0x0a3d70a3}, {0x26e978d5, 0x020c49ba},
     {0x3afb7e91, 0x0068db8b}, {0x0bcbe61d, 0x0014f8b5},
     {0x68c26139, 0x000431bd}, {0xae8d46a5, 0x0000d6bf},
     {0x22e90e21, 0x00002af3}, {0x3a2e9c6d, 0x00000897},
     {0x3ed61f49, 0x000001b7}};

// 存储用于 pow(5, k) 的除法测试表的条目，其中 k = 64 位无符号整数
template <typename T>
const divtest_table_entry<uint64_t> basic_data<T>::divtest_table_for_pow5_64[] =
    # 创建一个包含多个元组的数组，每个元组包含两个十六进制数
    {
        # 第一个元组
        0x0000000000000001,  # 十六进制数1
        0xffffffffffffffff    # 十六进制数-1
    },
    {
        # 第二个元组
        0xcccccccccccccccd,   # 十六进制数34028236692093846341
        0x3333333333333333    # 十六进制数133112935488063
    },
    {
        # 第三个元组
        0x8f5c28f5c28f5c29,  # 十六进制数10101010101010101
        0x0a3d70a3d70a3d70   # 十六进制数10000000000000000
    },
    # 其他元组依此类推
// 定义一个模板类的静态成员变量，存储 64 位浮点数的 dragonbox_pow10_significands
template <typename T>
const uint64_t basic_data<T>::dragonbox_pow10_significands_64[] = {
    // 64 位浮点数的 dragonbox_pow10_significands 数组
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

// 定义一个模板类的静态成员变量，存储 128 位浮点数的 dragonbox_pow10_significands
template <typename T>
const uint128_wrapper basic_data<T>::dragonbox_pow10_significands_128[] = {
    // 如果使用完整缓存 dragonbox，则存储 128 位浮点数的 dragonbox_pow10_significands 数组
    {0xff77b1fcbebcdc4f, 0x25e8e89c13bb0f7b},
    {0x9faacf3df73609b1, 0x77b191618c54e9ad},
    # 这些是一系列的十六进制数值，可能是用作密钥、哈希值或其他加密相关的数据
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
    {
        0xb2fe3f0b8599ef07, 0x861fa7e6dcb4aa16},
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
    }
    # 这部分代码是一系列的十六进制数对，可能是用作数据的密钥或者哈希值
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
    # 以下是一系列的十六进制数对，可能是用作密钥或哈希值
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
    # 以下是一系列的十六进制数对，可能是用于某种加密或编码算法的参数
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
    # 以下是一系列的十六进制数对，可能是密钥或哈希值
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
    # 以十六进制表示的一组数据
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
    # 以十六进制表示的数值，每个数值由两部分组成，分别是高位和低位
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
    {0x96769950b50d88f4, 0x131444800000000
    # 以十六进制表示的整数对，可能是某种数据的编码或者密钥
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
    # 以下是一系列的十六进制数对，可能是用作数据的密钥或者哈希值
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
    # 这些是一系列的十六进制数值，可能是用作密钥或哈希值
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
    {
        0xf0fdf2d3f3c30b9f, 0x656d44a2a11c51d5},  # 第一对十六进制数
        {0x969eb7c47859e743, 0x9f644ae5a4b1b325},  # 第二对十六进制数
        {0xbc4665b596706114, 0x873d5d9f0dde1fee},  # 第三对十六进制数
        {0xeb57ff22fc0c7959, 0xa90cb506d155a7ea},  # 第四对十六进制数
        {0x9316ff75dd87cbd8, 0x09a7f12442d588f2},  # 第五对十六进制数
        {0xb7dcbf5354e9bece, 0x0c11ed6d538aeb2f},  # 第六对十六进制数
        {0xe5d3ef282a242e81, 0x8f1668c8a86da5fa},  # 第七对十六进制数
        {0x8fa475791a569d10, 0xf96e017d694487bc},  # 第八对十六进制数
        {0xb38d92d760ec4455, 0x37c981dcc395a9ac},  # 第九对十六进制数
        {0xe070f78d3927556a, 0x85bbe253f47b1417},  # 第十对十六进制数
        {0x8c469ab843b89562, 0x93956d7478ccec8e},  # 第十一对十六进制数
        {0xaf58416654a6babb, 0x387ac8d1970027b2},  # 第十二对十六进制数
        {0xdb2e51bfe9d0696a, 0x06997b05fcc0319e},  # 第十三对十六进制数
        {0x88fcf317f22241e2, 0x441fece3bdf81f03},  # 第十四对十六进制数
        {0xab3c2fddeeaad25a, 0xd527e81cad7626c3},  # 第十五对十六进制数
        {0xd60b3bd56a5586f1, 0x8a71e223d8d3b074},  # 第十六对十六进制数
        {0x85c7056562757456, 0xf6872d5667844e49},  # 第十七对十六进制数
        {0xa738c6bebb12d16c, 0xb428f8ac016561db},  # 第十八对十六进制数
        {0xd106f86e69d785c7, 0xe13336d701beba52},  # 第十九对十六进制数
        {0x82a45b450226b39c, 0xecc0024661173473},  # 第二十对十六进制数
        {0xa34d721642b06084, 0x27f002d7f95d0190},  # 第二十一对十六进制数
        {0xcc20ce9bd35c78a5, 0x31ec038df7b441f4},  # 第二十二对十六进制数
        {0xff290242c83396ce, 0x7e67047175a15271},  # 第二十三对十六进制数
        {0x9f79a169bd203e41, 0x0f0062c6e984d386},  # 第二十四对十六进制数
        {0xc75809c42c684dd1, 0x52c07b78a3e60868},  # 第二十五对十六进制数
        {0xf92e0c3537826145, 0xa7709a56ccdf8a82},  # 第二十六对十六进制数
        {0x9bbcc7a142b17ccb, 0x88a66076400bb691},  # 第二十七对十六进制数
        {0xc2abf989935ddbfe, 0x6acff893d00ea435},  # 第二十八对十六进制数
        {0xf356f7ebf83552fe, 0x0583f6b8c4124d43},  # 第二十九对十六进制数
        {0x98165af37b2153de, 0xc3727a337a8b704a},  # 第三十对十六进制数
        {0xbe1bf1b059e9a8d6, 0x744f18c0592e4c5c},  # 第三十一对十六进制数
        {0xeda2ee1c7064130c, 0x1162def06f79df73},  # 第三十二对十六进制数
        {0x9485d4d1c63e8be7, 0x8addcb5645ac2ba8},  # 第三十三对十六进制数
        {0xb9a74a0637ce2ee1, 0x6d953e2bd7173692},  # 第三十四对十六进制数
        {0xe8111c87c5c1ba99, 0xc8fa8db6ccdd0437},  # 第三十五对十六进制数
        {0x910ab1d4db9914a0, 0x1d9c9892400a22a2},  # 第三十六对十六进制数
        {0xb54d5e4a127f59c8, 0x2503beb6d00cab4b},  # 第三十七对十六进制数
        {0xe2a0b5dc971f303a, 0x2e44ae64840fd61d},  # 第三十八对十六进制数
        {0x8da471a9de737e24, 0x5ceaecfed289e5d2},  # 第三十九对十六进制数
        {0xb10d8e1456105dad, 0x7425a83e872c5f47},  # 第四十对十六进制数
        {0xdd50f1996b947518, 0xd12f124e28f77719},  # 第四十一对十六进制数
        {0x8a5296ffe33cc92f, 0x82bd6b70d99aaa6f},  # 第四十二对十六进制数
        {0xace73cbfdc0bfb7b, 0x636cc64d1001550b},  # 第四十三对十六进制数
    }
    # 以下是一系列的十六进制数对，可能是密钥或哈希值
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
    # 这些是一系列的十六进制数值，可能是用作常量或者密钥
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
    # 这些看起来是一组十六进制数，可能是用作某种加密或哈希算法的输入或输出
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
#else
    {0xff77b1fcbebcdc4f, 0x25e8e89c13bb0f7b},  // 如果不满足条件，则使用这对值
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
#endif
};

#if !FMT_USE_FULL_CACHE_DRAGONBOX
template <typename T>
const uint64_t basic_data<T>::powers_of_5_64[] = {
    0x0000000000000001, 0x0000000000000005, 0x0000000000000019,  // 5的幂的64位表示
    0x000000000000007d, 0x0000000000000271, 0x0000000000000c35,
    0x0000000000003d09, 0x000000000001312d, 0x000000000005f5e1,
    0x00000000001dcd65, 0x00000000009502f9, 0x0000000002e90edd,
    0x000000000e8d4a51, 0x0000000048c27395, 0x000000016bcc41e9,
    0x000000071afd498d, 0x0000002386f26fc1, 0x000000b1a2bc2ec5,
    0x000003782dace9d9, 0x00001158e460913d, 0x000056bc75e2d631,
    0x0001b1ae4d6e2ef5, 0x000878678326eac9, 0x002a5a058fc295ed,
    0x00d3c21bcecceda1, 0x0422ca8b0a00a425, 0x14adf4b7320334b9};  // 5的幂的64位表示数组

template <typename T>
const uint32_t basic_data<T>::dragonbox_pow10_recovery_errors[] = {
    0x50001400, 0x54044100, 0x54014555, 0x55954415, 0x54115555, 0x00000001,  // 用于恢复10的幂的错误
    # 定义了一个包含多个十六进制数字的数组
    0x50000000, 0x00104000, 0x54010004, 0x05004001, 0x55555544, 0x41545555,
    0x54040551, 0x15445545, 0x51555514, 0x10000015, 0x00101100, 0x01100015,
    0x00000000, 0x00000000, 0x00000000, 0x00000000, 0x04450514, 0x45414110,
    0x55555145, 0x50544050, 0x15040155, 0x11054140, 0x50111514, 0x11451454,
    0x00400541, 0x00000000, 0x55555450, 0x10056551, 0x10054011, 0x55551014,
    0x69514555, 0x05151109, 0x00155555};
#endif

template <typename T>
const char basic_data<T>::foreground_color[] = "\x1b[38;2;";
// 前景色的 ANSI 转义序列

template <typename T>
const char basic_data<T>::background_color[] = "\x1b[48;2;";
// 背景色的 ANSI 转义序列

template <typename T> const char basic_data<T>::reset_color[] = "\x1b[0m";
// 重置颜色的 ANSI 转义序列

template <typename T> const wchar_t basic_data<T>::wreset_color[] = L"\x1b[0m";
// 宽字符版本的重置颜色的 ANSI 转义序列

template <typename T> const char basic_data<T>::signs[] = {0, '-', '+', ' '};
// 符号数组，包含了 0、-、+、空格

template <typename T>
const char basic_data<T>::left_padding_shifts[] = {31, 31, 0, 1, 0};
// 左填充位移数组

template <typename T>
const char basic_data<T>::right_padding_shifts[] = {0, 31, 0, 1, 0};
// 右填充位移数组

template <typename T> struct bits {
  static FMT_CONSTEXPR_DECL const int value =
      static_cast<int>(sizeof(T) * std::numeric_limits<unsigned char>::digits);
};
// 位数结构体，包含了位数的值

class fp;
template <int SHIFT = 0> fp normalize(fp value);
// 模板函数声明，用于规范化浮点数

// Lower (upper) boundary is a value half way between a floating-point value
// and its predecessor (successor). Boundaries have the same exponent as the
// value so only significands are stored.
struct boundaries {
  uint64_t lower;
  uint64_t upper;
};
// 边界结构体，存储浮点数的上下边界值

// A handmade floating-point number f * pow(2, e).
// 一个手工制作的浮点数 f * pow(2, e)
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

  // Default constructor, initializes f and e to 0
  fp() : f(0), e(0) {}
  // Constructor with specified f and e values
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
    // 如果 d 是标准化的2的幂（f == 0），且偏置指数大于1，则前驱更接近
    bool is_predecessor_closer = f == 0 && biased_e > 1;
    // 如果偏置指数不等于0，则将 f 加上隐含的位
    if (biased_e != 0)
      f += float_implicit_bit;
    else
      biased_e = 1;  // 亚标准数使用偏置指数1（最小指数）
    // 计算指数 e
    e = biased_e - exponent_bias - float_significand_size;
    // 返回前驱是否更接近
    return is_predecessor_closer;
  }

  // 如果 Float 类型不受支持，则将当前对象赋值为 fp()，并返回 false
  template <typename Float, FMT_ENABLE_IF(!is_supported_float<Float>::value)>
  bool assign(Float) {
    *this = fp();
    return false;
  }
// 标准化从 double 转换并乘以 (1 << SHIFT) 的值
template <int SHIFT> fp normalize(fp value) {
  // 处理次标准数
  const auto shifted_implicit_bit = fp::implicit_bit << SHIFT;
  while ((value.f & shifted_implicit_bit) == 0) {
    value.f <<= 1;
    --value.e;
  }
  // 减去 1 来考虑隐藏位
  const auto offset =
      fp::significand_size - fp::double_significand_size - SHIFT - 1;
  value.f <<= offset;
  value.e -= offset;
  return value;
}

// 比较操作符，判断两个 fp 对象是否相等
inline bool operator==(fp x, fp y) { return x.f == y.f && x.e == y.e; }

// 计算 lhs * rhs / pow(2, 64) 并四舍五入到最近的值
inline uint64_t multiply(uint64_t lhs, uint64_t rhs) {
#if FMT_USE_INT128
  auto product = static_cast<__uint128_t>(lhs) * rhs;
  auto f = static_cast<uint64_t>(product >> 64);
  return (static_cast<uint64_t>(product) & (1ULL << 63)) != 0 ? f + 1 : f;
#else
  // 计算意义位的 32 位部分
  uint64_t mask = (1ULL << 32) - 1;
  uint64_t a = lhs >> 32, b = lhs & mask;
  uint64_t c = rhs >> 32, d = rhs & mask;
  uint64_t ac = a * c, bc = b * c, ad = a * d, bd = b * d;
  // 计算结果的中间 64 位并四舍五入
  uint64_t mid = (bd >> 32) + (ad & mask) + (bc & mask) + (1U << 31);
  return ac + (ad >> 32) + (bc >> 32) + (mid >> 32);
#endif
}

// 重载 * 操作符，计算两个 fp 对象的乘积
inline fp operator*(fp x, fp y) { return {multiply(x.f, y.f), x.e + y.e + 64}; }

// 返回一个缓存的 10 的幂 `c_k = c_k.f * pow(2, c_k.e)`，使得其（二进制）指数满足 `min_exponent <= c_k.e <= min_exponent + 28`
// 返回缓存的指数和对应的10的幂
inline fp get_cached_power(int min_exponent, int& pow10_exponent) {
  // 定义位移量
  const int shift = 32;
  // 获取log10(2)的有效数字
  const auto significand = static_cast<int64_t>(data::log10_2_significand);
  // 计算缓存的指数
  int index = static_cast<int>(
      ((min_exponent + fp::significand_size - 1) * (significand >> shift) +
       ((int64_t(1) << shift) - 1))  // ceil
      >> 32                          // arithmetic shift
  );
  // 第一个（最小的）缓存的10的幂的十进制指数
  const int first_dec_exp = -348;
  // 缓存的10的幂中两个连续的十进制指数之间的差
  const int dec_exp_step = 8;
  index = (index - first_dec_exp - 1) / dec_exp_step + 1;
  // 计算10的幂的指数
  pow10_exponent = first_dec_exp + index * dec_exp_step;
  // 返回结果
  return {data::grisu_pow10_significands[index],
          data::grisu_pow10_exponents[index]};
}

// 如果uint128_t不可用，则使用简单的累加器来保存bigint::square中项的和
struct accumulator {
  uint64_t lower;
  uint64_t upper;

  accumulator() : lower(0), upper(0) {}
  explicit operator uint32_t() const { return static_cast<uint32_t>(lower); }

  // 累加操作
  void operator+=(uint64_t n) {
    lower += n;
    if (lower < n) ++upper;
  }
  // 右移操作
  void operator>>=(int shift) {
    assert(shift == 32);
    (void)shift;
    lower = (upper << 32) | (lower >> 32);
    upper >>= 32;
  }
};

class bigint {
 private:
  // bigint以bigits（大数字）数组的形式存储，索引为0的bigit是最低有效位
  using bigit = uint32_t;
  using double_bigit = uint64_t;
  enum { bigits_capacity = 32 };
  basic_memory_buffer<bigit, bigits_capacity> bigits_;
  int exp_;

  // 获取指定索引处的bigit
  bigit operator[](int index) const { return bigits_[to_unsigned(index)]; }
  bigit& operator[](int index) { return bigits_[to_unsigned(index)]; }

  // 定义bigit的位数
  static FMT_CONSTEXPR_DECL const int bigit_bits = bits<bigit>::value;

  friend struct formatter<bigint>;

  // 从指定索引处的bigit中减去另一个bigit，并更新借位
  void subtract_bigits(int index, bigit other, bigit& borrow) {
  // 计算两个大整数的差值，并考虑借位
  auto result = static_cast<double_bigit>((*this)[index]) - other - borrow;
  // 更新当前位置的大整数值
  (*this)[index] = static_cast<bigit>(result);
  // 更新借位值
  borrow = static_cast<bigit>(result >> (bigit_bits * 2 - 1));
}

// 移除大整数前导零
void remove_leading_zeros() {
  // 获取大整数的位数
  int num_bigits = static_cast<int>(bigits_.size()) - 1;
  // 移除前导零
  while (num_bigits > 0 && (*this)[num_bigits] == 0) --num_bigits;
  bigits_.resize(to_unsigned(num_bigits + 1));
}

// 计算 *this -= other，假设大整数已对齐且 *this >= other
void subtract_aligned(const bigint& other) {
  // 断言大整数已对齐
  FMT_ASSERT(other.exp_ >= exp_, "unaligned bigints");
  // 断言 *this >= other
  FMT_ASSERT(compare(*this, other) >= 0, "");
  // 初始化借位值
  bigit borrow = 0;
  int i = other.exp_ - exp_;
  // 逐位相减
  for (size_t j = 0, n = other.bigits_.size(); j != n; ++i, ++j)
    subtract_bigits(i, other.bigits_[j], borrow);
  // 处理剩余的借位
  while (borrow > 0) subtract_bigits(i, 0, borrow);
  // 移除前导零
  remove_leading_zeros();
}

// 乘以 32 位无符号整数
void multiply(uint32_t value) {
  // 将 value 转换为双位大整数
  const double_bigit wide_value = value;
  // 初始化进位值
  bigit carry = 0;
  // 逐位相乘
  for (size_t i = 0, n = bigits_.size(); i < n; ++i) {
    double_bigit result = bigits_[i] * wide_value + carry;
    bigits_[i] = static_cast<bigit>(result);
    carry = static_cast<bigit>(result >> bigit_bits);
  }
  // 处理最高位的进位
  if (carry != 0) bigits_.push_back(carry);
}

// 乘以 64 位无符号整数
void multiply(uint64_t value) {
  // 定义掩码和双位大整数的低位和高位
  const bigit mask = ~bigit(0);
  const double_bigit lower = value & mask;
  const double_bigit upper = value >> bigit_bits;
  double_bigit carry = 0;
  // 逐位相乘
  for (size_t i = 0, n = bigits_.size(); i < n; ++i) {
    double_bigit result = bigits_[i] * lower + (carry & mask);
    carry =
        bigits_[i] * upper + (result >> bigit_bits) + (carry >> bigit_bits);
    bigits_[i] = static_cast<bigit>(result);
  }
  // 处理剩余的进位
  while (carry != 0) {
    bigits_.push_back(carry & mask);
    carry >>= bigit_bits;
  }
}
  }
}

public:
// 默认构造函数，将指数初始化为0
bigint() : exp_(0) {}
// 显式构造函数，将无符号64位整数赋值给bigint对象
explicit bigint(uint64_t n) { assign(n); }
// 析构函数，断言bigits_的容量不超过bigits_capacity
~bigint() { assert(bigits_.capacity() <= bigits_capacity); }

// 禁用拷贝构造函数和赋值运算符重载
bigint(const bigint&) = delete;
void operator=(const bigint&) = delete;

// 将另一个bigint对象的值赋给当前对象
void assign(const bigint& other) {
  auto size = other.bigits_.size();
  bigits_.resize(size);
  auto data = other.bigits_.data();
  std::copy(data, data + size, make_checked(bigits_.data(), size));
  exp_ = other.exp_;
}

// 将无符号64位整数赋值给bigint对象
void assign(uint64_t n) {
  size_t num_bigits = 0;
  do {
    bigits_[num_bigits++] = n & ~bigit(0);
    n >>= bigit_bits;
  } while (n != 0);
  bigits_.resize(num_bigits);
  exp_ = 0;
}

// 返回bigint对象的位数
int num_bigits() const { return static_cast<int>(bigits_.size()) + exp_; }

// 左移运算符重载
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

// 乘法运算符重载
template <typename Int> bigint& operator*=(Int value) {
  FMT_ASSERT(value > 0, "");
  multiply(uint32_or_64_or_128_t<Int>(value));
  return *this;
}

// 比较两个bigint对象的大小
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
    // 返回整数0
    return 0;
  }

  // 返回比较(lhs1 + lhs2, rhs)的结果
  friend int add_compare(const bigint& lhs1, const bigint& lhs2,
                         const bigint& rhs) {
    // 计算左操作数的位数
    int max_lhs_bigits = (std::max)(lhs1.num_bigits(), lhs2.num_bigits());
    // 获取右操作数的位数
    int num_rhs_bigits = rhs.num_bigits();
    // 如果左操作数的位数加1小于右操作数的位数，则返回-1
    if (max_lhs_bigits + 1 < num_rhs_bigits) return -1;
    // 如果左操作数的位数大于右操作数的位数，则返回1
    if (max_lhs_bigits > num_rhs_bigits) return 1;
    // 定义一个lambda函数，用于获取指定位置的位数
    auto get_bigit = [](const bigint& n, int i) -> bigit {
      return i >= n.exp_ && i < n.num_bigits() ? n[i - n.exp_] : 0;
    };
    // 定义一个双位数的借位
    double_bigit borrow = 0;
    // 获取最小的指数
    int min_exp = (std::min)((std::min)(lhs1.exp_, lhs2.exp_), rhs.exp_);
    // 从右操作数的位数-1开始循环到最小指数
    for (int i = num_rhs_bigits - 1; i >= min_exp; --i) {
      // 计算左操作数的位数之和
      double_bigit sum =
          static_cast<double_bigit>(get_bigit(lhs1, i)) + get_bigit(lhs2, i);
      // 获取右操作数的位数
      bigit rhs_bigit = get_bigit(rhs, i);
      // 如果左操作数的位数之和大于右操作数的位数加上借位，则返回1
      if (sum > rhs_bigit + borrow) return 1;
      // 更新借位
      borrow = rhs_bigit + borrow - sum;
      // 如果借位大于1，则返回-1
      if (borrow > 1) return -1;
      // 左移位数
      borrow <<= bigit_bits;
    }
    // 如果借位不为0，则返回-1，否则返回0
    return borrow != 0 ? -1 : 0;
  }

  // 将pow(10, exp)赋值给这个bigint
  void assign_pow10(int exp) {
    // 断言指数大于等于0
    assert(exp >= 0);
    // 如果指数为0，则赋值为1
    if (exp == 0) return assign(1);
    // 找到最高位
    int bitmask = 1;
    while (exp >= bitmask) bitmask <<= 1;
    bitmask >>= 1;
    // 计算pow(10, exp) = pow(5, exp) * pow(2, exp)
    // 首先计算pow(5, exp)通过重复平方和乘法
    assign(5);
    bitmask >>= 1;
    while (bitmask != 0) {
      square();
      if ((exp & bitmask) != 0) *this *= 5;
      bitmask >>= 1;
    }
    *this <<= exp;  // 通过移位乘以pow(2, exp)
  }

  // 计算平方
  void square() {
    // 创建一个新的bigint对象
    basic_memory_buffer<bigit, bigits_capacity> n(std::move(bigits_));
    // 获取位数
    int num_bigits = static_cast<int>(bigits_.size());
    // 计算结果的位数
    int num_result_bigits = 2 * num_bigits;
    bigits_.resize(to_unsigned(num_result_bigits));
    // 定义累加器类型
    using accumulator_t = conditional_t<FMT_USE_INT128, uint128_t, accumulator>;
    auto sum = accumulator_t();
    // 遍历大整数的每个 bigit 位
    for (int bigit_index = 0; bigit_index < num_bigits; ++bigit_index) {
      // 计算结果中 bigit_index 位置的 bigit，通过添加交叉乘积项 n[i] * n[j] 来实现，其中 i + j == bigit_index
      for (int i = 0, j = bigit_index; j >= 0; ++i, --j) {
        // 大多数项被乘以两次，可以在将来进行优化
        sum += static_cast<double_bigit>(n[i]) * n[j];
      }
      // 将计算得到的结果赋值给结果的对应位置
      (*this)[bigit_index] = static_cast<bigit>(sum);
      // 计算进位
      sum >>= bits<bigit>::value;
    }
    // 对结果的上半部分执行相同的操作
    for (int bigit_index = num_bigits; bigit_index < num_result_bigits;
         ++bigit_index) {
      for (int j = num_bigits - 1, i = bigit_index - j; i < num_bigits;)
        sum += static_cast<double_bigit>(n[i++]) * n[j--];
      (*this)[bigit_index] = static_cast<bigit>(sum);
      sum >>= bits<bigit>::value;
    }
    // 减少结果的 bigit 数量
    --num_result_bigits;
    // 移除结果中的前导零
    remove_leading_zeros();
    // 指数乘以 2
    exp_ *= 2;
  }

  // 如果这个大整数的指数比其他大整数的指数大，添加尾随零以使指数相等。这简化了一些操作，如减法。
  void align(const bigint& other) {
    // 计算指数之间的差异
    int exp_difference = exp_ - other.exp_;
    // 如果差异小于等于0，则返回
    if (exp_difference <= 0) return;
    // 调整 bigit 数组的大小
    int num_bigits = static_cast<int>(bigits_.size());
    bigits_.resize(to_unsigned(num_bigits + exp_difference));
    // 将元素向后移动以添加尾随零
    for (int i = num_bigits - 1, j = i + exp_difference; i >= 0; --i, --j)
      bigits_[j] = bigits_[i];
    // 使用未初始化的值填充数组的前 exp_difference 个元素
    std::uninitialized_fill_n(bigits_.data(), exp_difference, 0);
    // 调整指数
    exp_ -= exp_difference;
  }

  // 将这个大整数除以除数，将余数赋给这个大整数，并返回商
  int divmod_assign(const bigint& divisor) {
    // 断言这个大整数不等于除数
    FMT_ASSERT(this != &divisor, "");
    // 如果这个大整数小于除数，则返回0
    if (compare(*this, divisor) < 0) return 0;
    // 断言除数的最高位不为0
    FMT_ASSERT(divisor.bigits_[divisor.bigits_.size() - 1u] != 0, "");
    // 调整这个大整数，使其与除数的指数相等
    align(divisor);
    // 初始化商
    int quotient = 0;
    // 执行除法操作，计算商
    do {
      subtract_aligned(divisor);
      ++quotient;
    } while (compare(*this, divisor) >= 0);
    # 使用do-while循环，当this和divisor比较的结果大于等于0时继续执行循环
    return quotient;
  }
    # 返回商
// 结束枚举定义
};

// 定义枚举类型 round_direction，包括未知、向上取整、向下取整
enum class round_direction { unknown, up, down };

// 给定除数（通常是10的幂）、余数 = v % divisor 以及误差，返回应该向上取整、向下取整，或者由于误差无法确定取整方向
// 误差应该小于除数的一半
inline round_direction get_round_direction(uint64_t divisor, uint64_t remainder,
                                           uint64_t error) {
  FMT_ASSERT(remainder < divisor, "");  // divisor - remainder 不会溢出。
  FMT_ASSERT(error < divisor, "");      // divisor - error 不会溢出。
  FMT_ASSERT(error < divisor - error, "");  // error * 2 不会溢出。
  // 如果 (remainder + error) * 2 <= divisor，则向下取整。
  if (remainder <= divisor - remainder && error * 2 <= divisor - remainder * 2)
    return round_direction::down;
  // 如果 (remainder - error) * 2 >= divisor，则向上取整。
  if (remainder >= error &&
      remainder - error >= divisor - (remainder - error)) {
    return round_direction::up;
  }
  return round_direction::unknown;
}

// 命名空间 digits
namespace digits {
// 定义枚举类型 result
enum result {
  more,  // 生成更多数字。
  done,  // 完成数字生成。
  error  // 由于错误取消数字生成。
};
}

// 使用 Grisu 数字生成算法生成输出。
// error: 区域 (lower, upper) 的大小，在该区域外的数字肯定不会取整为 value（Grisu3 中的 Delta）。
template <typename Handler>
FMT_ALWAYS_INLINE digits::result grisu_gen_digits(fp value, uint64_t error,
                                                  int& exp, Handler& handler) {
  // 计算 2 的负指数次方，构造一个 fp 对象
  const fp one(1ULL << -value.e, value.e);
  // 计算 scaled value 的整数部分，即 p1 in Grisu
  // 由于归一化，它不可能为零，因为它包含两个最高位设置的 64 位数的乘积 - 1，最多向右移动 60 位。
  auto integral = static_cast<uint32_t>(value.f >> -one.e);
  FMT_ASSERT(integral != 0, "");
  FMT_ASSERT(integral == value.f >> -one.e, "");
  // 计算 scaled value 的小数部分，即 p2 in Grisu
  uint64_t fractional = value.f & (one.f - 1);
  // 计算 kappa in Grisu，即整数部分的位数
  exp = count_digits(integral);
  // 除以 10 以防止溢出
  auto result = handler.on_start(data::powers_of_10_64[exp - 1] << -one.e,
                                 value.f / 10, error * 10, exp);
  if (result != digits::more) return result;
  // 为整数部分生成数字。这可能产生多达 10 位数字。
  do {
    uint32_t digit = 0;
    auto divmod_integral = [&](uint32_t divisor) {
      digit = integral / divisor;
      integral %= divisor;
    };
    // Milo Yip 的优化减少了每次迭代的整数除法次数。
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
    // 计算余数，将整数部分左移 -one.e 位后加上小数部分
    auto remainder = (static_cast<uint64_t>(integral) << -one.e) + fractional;
    // 调用处理程序处理当前数字，并返回结果
    result = handler.on_digit(static_cast<char>('0' + digit),
                              data::powers_of_10_64[exp] << -one.e, remainder,
                              error, exp, true);
    // 如果处理结果不是更多数字，则返回结果
    if (result != digits::more) return result;
  } while (exp > 0);
  // 生成小数部分的数字
  for (;;) {
    // 小数部分乘以 10
    fractional *= 10;
    // 误差乘以 10
    error *= 10;
    // 获取当前位的数字
    char digit = static_cast<char>('0' + (fractional >> -one.e));
    // 更新小数部分
    fractional &= one.f - 1;
    // 指数减一
    --exp;
    // 调用处理程序处理当前数字，并返回结果
    result = handler.on_digit(digit, one.f, fractional, error, exp, false);
    // 如果处理结果不是更多数字，则返回结果
    if (result != digits::more) return result;
  }
// 固定精度数字处理程序的结构体
struct fixed_handler {
  char* buf; // 存储数字的缓冲区
  int size; // 缓冲区大小
  int precision; // 精度
  int exp10; // 10的指数
  bool fixed; // 是否为固定格式

  // 处理开始时的回调函数
  digits::result on_start(uint64_t divisor, uint64_t remainder, uint64_t error,
                          int& exp) {
    // 非固定格式至少需要一个数字，不需要精度调整
    if (!fixed) return digits::more;
    // 根据指数调整固定精度，因为它是相对于小数点的
    precision += exp + exp10;
    // 检查精度是否仅由前导零满足，例如 format("{:.2f}", 0.001) 会得到 "0.00" 而不生成任何数字
    if (precision > 0) return digits::more;
    if (precision < 0) return digits::done;
    auto dir = get_round_direction(divisor, remainder, error);
    if (dir == round_direction::unknown) return digits::error;
    buf[size++] = dir == round_direction::up ? '1' : '0';
    return digits::done;
  }

  // 处理数字时的回调函数
  digits::result on_digit(char digit, uint64_t divisor, uint64_t remainder,
                          uint64_t error, int, bool integral) {
    FMT_ASSERT(remainder < divisor, "");
    buf[size++] = digit;
    if (!integral && error >= remainder) return digits::error;
    if (size < precision) return digits::more;
    if (!integral) {
      // 检查 error * 2 < divisor 并防止溢出
      // 对于整数部分不需要进行检查，因为 error = 1 并且 divisor > (1 << 32)
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
    # 返回变量 digits 的值，并标记为已完成
    return digits::done;
  }
};

// Dragonbox算法的实现：https://github.com/jk-jeon/dragonbox。
namespace dragonbox {
// 计算两个64位无符号整数的乘积的128位结果。
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
#endif
}

// 计算两个64位无符号整数的乘积的高64位。
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

// 计算一个64位无符号整数和一个128位无符号整数的乘积的高64位。
FMT_SAFEBUFFERS inline uint64_t umul192_upper64(uint64_t x, uint128_wrapper y)
    FMT_NOEXCEPT {
  uint128_wrapper g0 = umul128(x, y.high());
  g0 += umul128_upper64(x, y.low());
  return g0.high();
}

// 计算一个32位无符号整数和一个64位无符号整数的乘积的高32位。
inline uint32_t umul96_upper32(uint32_t x, uint64_t y) FMT_NOEXCEPT {
  return static_cast<uint32_t>(umul128_upper64(x, y));
}
// 计算一个 64 位无符号整数和一个 128 位无符号整数的乘积的中间 64 位
FMT_SAFEBUFFERS inline uint64_t umul192_middle64(uint64_t x, uint128_wrapper y)
    FMT_NOEXCEPT {
  // 计算 x 乘以 y 的高 64 位
  uint64_t g01 = x * y.high();
  // 计算 x 乘以 y 的低 64 位的高 64 位
  uint64_t g10 = umul128_upper64(x, y.low());
  // 返回 g01 和 g10 的和
  return g01 + g10;
}

// 计算一个 32 位无符号整数和一个 64 位无符号整数的乘积的低 64 位
inline uint64_t umul96_lower64(uint32_t x, uint64_t y) FMT_NOEXCEPT {
  // 返回 x 乘以 y 的结果
  return x * y;
}

// 计算 pow(2, e) 的 floor(log10())，其中 e 在 [-1700, 1700] 范围内，使用 https://fmt.dev/papers/Grisu-Exact.pdf#page=5 中的方法，第 3.4 节
inline int floor_log10_pow2(int e) FMT_NOEXCEPT {
  // 断言 e 在合理范围内
  FMT_ASSERT(e <= 1700 && e >= -1700, "too large exponent");
  // 定义位移量
  const int shift = 22;
  // 返回计算结果
  return (e * static_cast<int>(data::log10_2_significand >> (64 - shift))) >>
         shift;
}

// 各种快速对数计算
inline int floor_log2_pow10(int e) FMT_NOEXCEPT {
  // 断言 e 在合理范围内
  FMT_ASSERT(e <= 1233 && e >= -1233, "too large exponent");
  // 定义常量
  const uint64_t log2_10_integer_part = 3;
  const uint64_t log2_10_fractional_digits = 0x5269e12f346e2bf9;
  const int shift_amount = 19;
  // 返回计算结果
  return (e * static_cast<int>(
                  (log2_10_integer_part << shift_amount) |
                  (log2_10_fractional_digits >> (64 - shift_amount)))) >>
         shift_amount;
}
inline int floor_log10_pow2_minus_log10_4_over_3(int e) FMT_NOEXCEPT {
  // 断言 e 在合理范围内
  FMT_ASSERT(e <= 1700 && e >= -1700, "too large exponent");
  // 定义常量
  const uint64_t log10_4_over_3_fractional_digits = 0x1ffbfc2bbc780375;
  const int shift_amount = 22;
  // 返回计算结果
  return (e * static_cast<int>(data::log10_2_significand >>
                               (64 - shift_amount)) -
          static_cast<int>(log10_4_over_3_fractional_digits >>
                           (64 - shift_amount))) >>
         shift_amount;
}

// 如果 x 可以被 pow(2, exp) 整除，则返回 true
// 检查 x 是否可以被 2 的 exp 次方整除，返回布尔值
inline bool divisible_by_power_of_2(uint32_t x, int exp) FMT_NOEXCEPT {
  // 断言 exp 大于等于 1
  FMT_ASSERT(exp >= 1, "");
  // 断言 x 不等于 0
  FMT_ASSERT(x != 0, "");
#ifdef FMT_BUILTIN_CTZ
  // 如果支持内建的计算 x 的二进制表示中末尾 0 的个数的函数，则使用该函数判断是否可以被 2 的 exp 次方整除
  return FMT_BUILTIN_CTZ(x) >= exp;
#else
  // 否则，通过位运算判断是否可以被 2 的 exp 次方整除
  return exp < num_bits<uint32_t>() && x == ((x >> exp) << exp);
#endif
}
// 与上一个函数类似，针对 64 位整数
inline bool divisible_by_power_of_2(uint64_t x, int exp) FMT_NOEXCEPT {
  FMT_ASSERT(exp >= 1, "");
  FMT_ASSERT(x != 0, "");
#ifdef FMT_BUILTIN_CTZLL
  return FMT_BUILTIN_CTZLL(x) >= exp;
#else
  return (exp < num_bits<uint64_t>()) && x == ((x >> exp) << exp);
#endif
}

// 返回 x 是否可以被 5 的 exp 次方整除
inline bool divisible_by_power_of_5(uint32_t x, int exp) FMT_NOEXCEPT {
  // 断言 exp 小于等于 10
  FMT_ASSERT(exp <= 10, "too large exponent");
  // 判断 x 是否可以被 5 的 exp 次方整除
  return x * data::divtest_table_for_pow5_32[exp].mod_inv <=
         data::divtest_table_for_pow5_32[exp].max_quotient;
}
// 与上一个函数类似，针对 64 位整数
inline bool divisible_by_power_of_5(uint64_t x, int exp) FMT_NOEXCEPT {
  FMT_ASSERT(exp <= 23, "too large exponent");
  return x * data::divtest_table_for_pow5_64[exp].mod_inv <=
         data::divtest_table_for_pow5_64[exp].max_quotient;
}

// 将 n 替换为 floor(n / pow(5, N))，返回 n 是否可以被 5 的 N 次方整除
// 前提条件：n <= 2 * pow(5, N + 1)
template <int N>
bool check_divisibility_and_divide_by_pow5(uint32_t& n) FMT_NOEXCEPT {
  // 预定义的一些信息
  static constexpr struct {
    uint32_t magic_number;
    int bits_for_comparison;
    uint32_t threshold;
    int shift_amount;
  } infos[] = {{0xcccd, 16, 0x3333, 18}, {0xa429, 8, 0x0a, 20}};
  // 获取对应于 N 的信息
  constexpr auto info = infos[N - 1];
  // 将 n 乘以魔数
  n *= info.magic_number;
  // 计算比较掩码
  const uint32_t comparison_mask = (1u << info.bits_for_comparison) - 1;
  // 判断 n 是否可以被 5 的 N 次方整除
  bool result = (n & comparison_mask) <= info.threshold;
  // 右移 n
  n >>= info.shift_amount;
  return result;
}

// 计算小整数 n 除以 10 的 N 次方的结果
// 前提条件：n <= pow(10, N + 1)
template <int N> uint32_t small_division_by_pow10(uint32_t n) FMT_NOEXCEPT {
  // 预定义的一些信息
  static constexpr struct {
    uint32_t magic_number;
    int shift_amount;
    // 定义一个结构体数组infos，包含divisor_times_10、shift_amount和magic_number三个成员
    } infos[] = {{0xcccd, 19, 100}, {0xa3d8, 22, 1000}};
    // 使用constexpr关键字定义一个常量info，其值为infos数组中索引为N-1的元素
    constexpr auto info = infos[N - 1];
    // 使用FMT_ASSERT宏来断言n是否小于等于info中的divisor_times_10，如果不满足则输出错误信息"n is too large"
    FMT_ASSERT(n <= info.divisor_times_10, "n is too large");
    // 返回n乘以info中的magic_number再右移info中的shift_amount位的结果
    return n * info.magic_number >> info.shift_amount;
// 计算 floor(n / 10^(kappa + 1)) (float)
inline uint32_t divide_by_10_to_kappa_plus_1(uint32_t n) FMT_NOEXCEPT {
  return n / float_info<float>::big_divisor;
}
// 计算 floor(n / 10^(kappa + 1)) (double)
inline uint64_t divide_by_10_to_kappa_plus_1(uint64_t n) FMT_NOEXCEPT {
  return umul128_upper64(n, 0x83126e978d4fdf3c) >> 9;
}

// 使用 pow10 缓存的各种子程序
template <class T> struct cache_accessor;

template <> struct cache_accessor<float> {
  using carrier_uint = float_info<float>::carrier_uint;
  using cache_entry_type = uint64_t;

  // 获取缓存的 10 的幂次方
  static uint64_t get_cached_power(int k) FMT_NOEXCEPT {
    FMT_ASSERT(k >= float_info<float>::min_k && k <= float_info<float>::max_k,
               "k is out of range");
    return data::dragonbox_pow10_significands_64[k - float_info<float>::min_k];
  }

  // 计算乘法
  static carrier_uint compute_mul(carrier_uint u,
                                  const cache_entry_type& cache) FMT_NOEXCEPT {
    return umul96_upper32(u, cache);
  }

  // 计算差值
  static uint32_t compute_delta(const cache_entry_type& cache,
                                int beta_minus_1) FMT_NOEXCEPT {
    return static_cast<uint32_t>(cache >> (64 - 1 - beta_minus_1));
  }

  // 计算乘法奇偶性
  static bool compute_mul_parity(carrier_uint two_f,
                                 const cache_entry_type& cache,
                                 int beta_minus_1) FMT_NOEXCEPT {
    FMT_ASSERT(beta_minus_1 >= 1, "");
    FMT_ASSERT(beta_minus_1 < 64, "");

    return ((umul96_lower64(two_f, cache) >> (64 - beta_minus_1)) & 1) != 0;
  }

  // 计算短区间情况下的左端点
  static carrier_uint compute_left_endpoint_for_shorter_interval_case(
      const cache_entry_type& cache, int beta_minus_1) FMT_NOEXCEPT {
  // 将缓存值减去右移后的值，再右移得到结果
  return static_cast<carrier_uint>(
      (cache - (cache >> (float_info<float>::significand_bits + 2))) >>
      (64 - float_info<float>::significand_bits - 1 - beta_minus_1));
}

// 计算较短区间情况下的右端点
static carrier_uint compute_right_endpoint_for_shorter_interval_case(
    const cache_entry_type& cache, int beta_minus_1) FMT_NOEXCEPT {
  // 将缓存值加上右移后的值，再右移得到结果
  return static_cast<carrier_uint>(
      (cache + (cache >> (float_info<float>::significand_bits + 1))) >>
      (64 - float_info<float>::significand_bits - 1 - beta_minus_1));
}

// 计算较短区间情况下的四舍五入值
static carrier_uint compute_round_up_for_shorter_interval_case(
    const cache_entry_type& cache, int beta_minus_1) FMT_NOEXCEPT {
  // 将缓存值右移得到结果，再加1再除以2
  return (static_cast<carrier_uint>(
              cache >>
              (64 - float_info<float>::significand_bits - 2 - beta_minus_1)) +
          1) /
         2;
}
};

// 特化模板，用于双精度浮点数的缓存访问
template <> struct cache_accessor<double> {
  // 使用 float_info<double>::carrier_uint 定义 carrier_uint 类型
  using carrier_uint = float_info<double>::carrier_uint;
  // 使用 uint128_wrapper 定义 cache_entry_type 类型
  using cache_entry_type = uint128_wrapper;

  // 获取缓存中的指定幂次的值
  static uint128_wrapper get_cached_power(int k) FMT_NOEXCEPT {
    // 断言 k 的范围在 float_info<double>::min_k 和 float_info<double>::max_k 之间
    FMT_ASSERT(k >= float_info<double>::min_k && k <= float_info<double>::max_k,
               "k is out of range");

    // 如果使用完整的缓存 dragonbox_pow10_significands_128
    return data::dragonbox_pow10_significands_128[k -
                                                  float_info<double>::min_k];
#else
    // 压缩比例
    static const int compression_ratio = 27;

    // 计算基本索引
    int cache_index = (k - float_info<double>::min_k) / compression_ratio;
    int kb = cache_index * compression_ratio + float_info<double>::min_k;
    int offset = k - kb;

    // 获取基本缓存
    uint128_wrapper base_cache =
        data::dragonbox_pow10_significands_128[cache_index];
    if (offset == 0) return base_cache;

    // 计算所需的位移量
    int alpha = floor_log2_pow10(kb + offset) - floor_log2_pow10(kb) - offset;
    FMT_ASSERT(alpha > 0 && alpha < 64, "shifting error detected");

    // 尝试恢复真实的缓存
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

    // 获取误差
    int error_idx = (k - float_info<double>::min_k) / 16;
    uint32_t error = (data::dragonbox_pow10_recovery_errors[error_idx] >>
                      ((k - float_info<double>::min_k) % 16) * 2) &
                     0x3;
    // 添加错误返回
    // 使用断言确保 recovered_cache.low() + error 大于等于 recovered_cache.low()
    FMT_ASSERT(recovered_cache.low() + error >= recovered_cache.low(), "");
    // 返回一个包含 recovered_cache.high() 和 recovered_cache.low() + error 的元组
    return {recovered_cache.high(), recovered_cache.low() + error};
#endif
  }

  // 计算乘法的结果的高 64 位
  static carrier_uint compute_mul(carrier_uint u,
                                  const cache_entry_type& cache) FMT_NOEXCEPT {
    return umul192_upper64(u, cache);
  }

  // 计算 delta 值
  static uint32_t compute_delta(cache_entry_type const& cache,
                                int beta_minus_1) FMT_NOEXCEPT {
    return static_cast<uint32_t>(cache.high() >> (64 - 1 - beta_minus_1));
  }

  // 计算乘法的奇偶性
  static bool compute_mul_parity(carrier_uint two_f,
                                 const cache_entry_type& cache,
                                 int beta_minus_1) FMT_NOEXCEPT {
    FMT_ASSERT(beta_minus_1 >= 1, "");
    FMT_ASSERT(beta_minus_1 < 64, "");

    return ((umul192_middle64(two_f, cache) >> (64 - beta_minus_1)) & 1) != 0;
  }

  // 计算短区间情况下的左端点
  static carrier_uint compute_left_endpoint_for_shorter_interval_case(
      const cache_entry_type& cache, int beta_minus_1) FMT_NOEXCEPT {
    return (cache.high() -
            (cache.high() >> (float_info<double>::significand_bits + 2))) >>
           (64 - float_info<double>::significand_bits - 1 - beta_minus_1);
  }

  // 计算短区间情况下的右端点
  static carrier_uint compute_right_endpoint_for_shorter_interval_case(
      const cache_entry_type& cache, int beta_minus_1) FMT_NOEXCEPT {
    return (cache.high() +
            (cache.high() >> (float_info<double>::significand_bits + 1))) >>
           (64 - float_info<double>::significand_bits - 1 - beta_minus_1);
  }

  // 计算短区间情况下的四舍五入值
  static carrier_uint compute_round_up_for_shorter_interval_case(
      const cache_entry_type& cache, int beta_minus_1) FMT_NOEXCEPT {
    return ((cache.high() >>
             (64 - float_info<double>::significand_bits - 2 - beta_minus_1)) +
            1) /
           2;
  }
};

// 各种整数检查
template <class T>
// 检查指数是否在短区间左端点的整数范围内
bool is_left_endpoint_integer_shorter_interval(int exponent) FMT_NOEXCEPT {
  return exponent >=
             float_info<
                 T>::case_shorter_interval_left_endpoint_lower_threshold &&
         exponent <=
             float_info<T>::case_shorter_interval_left_endpoint_upper_threshold;
}
// 检查端点是否为整数
template <class T>
bool is_endpoint_integer(typename float_info<T>::carrier_uint two_f,
                         int exponent, int minus_k) FMT_NOEXCEPT {
  if (exponent < float_info<T>::case_fc_pm_half_lower_threshold) return false;
  // 对于 k >= 0。
  if (exponent <= float_info<T>::case_fc_pm_half_upper_threshold) return true;
  // 对于 k < 0。
  if (exponent > float_info<T>::divisibility_check_by_5_threshold) return false;
  return divisible_by_power_of_5(two_f, minus_k);
}

template <class T>
bool is_center_integer(typename float_info<T>::carrier_uint two_f, int exponent,
                       int minus_k) FMT_NOEXCEPT {
  // 5 的指数为负数。
  if (exponent > float_info<T>::divisibility_check_by_5_threshold) return false;
  if (exponent > float_info<T>::case_fc_upper_threshold)
    return divisible_by_power_of_5(two_f, minus_k);
  // 两个指数都是非负数。
  if (exponent >= float_info<T>::case_fc_lower_threshold) return true;
  // 2 的指数为负数。
  return divisible_by_power_of_2(two_f, minus_k - exponent + 1);
}

// 从 n 中移除尾随的零并返回移除的零的数量（浮点数）
FMT_ALWAYS_INLINE int remove_trailing_zeros(uint32_t& n) FMT_NOEXCEPT {
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
    # 将 n 乘以 mod_inv2
    n *= mod_inv2;
  }
  # 如果 s 小于 t 并且 n 乘以 mod_inv1 小于等于 max_quotient1
  if (s < t && n * mod_inv1 <= max_quotient1) {
    # 将 n 乘以 mod_inv1
    n *= mod_inv1;
    # s 自增1
    ++s;
  }
  # 将 n 右移 s 位
  n >>= s;
  # 返回 s
  return s;
// 移除数字末尾的零并返回移除的零的数量（双精度）
FMT_ALWAYS_INLINE int remove_trailing_zeros(uint64_t& n) FMT_NOEXCEPT {
#ifdef FMT_BUILTIN_CTZLL
  // 使用内建函数获取末尾零的数量
  int t = FMT_BUILTIN_CTZLL(n);
#else
  // 使用自定义函数获取末尾零的数量
  int t = ctzll(n);
#endif
  // 如果末尾零的数量大于双精度浮点数的最大末尾零数量，则将其设置为最大末尾零数量
  if (t > float_info<double>::max_trailing_zeros)
    t = float_info<double>::max_trailing_zeros;
  // 除以 10^8 并将结果缩减为 32 位
  // 由于 ret_value.significand <= (2^64 - 1) / 1000 < 10^17，
  // 商和余数都应该适合于 32 位

  const uint32_t mod_inv1 = 0xcccccccd;
  const uint32_t max_quotient1 = 0x33333333;
  const uint64_t mod_inv8 = 0xc767074b22e90e21;
  const uint64_t max_quotient8 = 0x00002af31dc46118;

  // 如果数字可以被 1'0000'0000 整除，则使用商
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

  // 否则，使用余数
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
  }
}
  # 返回整数 4
  return 4;
}
# remainder 乘以 mod_inv1
remainder *= mod_inv1;

# 如果 t 等于 5 或者 remainder 乘以 mod_inv1 大于 max_quotient1
if (t == 5 || remainder * mod_inv1 > max_quotient1) {
  # 计算 n 的值
  n = (remainder >> 5) + quotient * 1000ull;
  # 返回整数 5
  return 5;
}
# remainder 乘以 mod_inv1
remainder *= mod_inv1;

# 如果 t 等于 6 或者 remainder 乘以 mod_inv1 大于 max_quotient1
if (t == 6 || remainder * mod_inv1 > max_quotient1) {
  # 计算 n 的值
  n = (remainder >> 6) + quotient * 100ull;
  # 返回整数 6
  return 6;
}
# remainder 乘以 mod_inv1
remainder *= mod_inv1;

# 计算 n 的值
n = (remainder >> 7) + quotient * 10ull;
# 返回整数 7
return 7;
// 较短区间情况的主要算法
template <class T>
FMT_ALWAYS_INLINE FMT_SAFEBUFFERS decimal_fp<T> shorter_interval_case(
    int exponent) FMT_NOEXCEPT {
  decimal_fp<T> ret_value;
  // 计算 k 和 beta
  const int minus_k = floor_log10_pow2_minus_log10_4_over_3(exponent);
  const int beta_minus_1 = exponent + floor_log2_pow10(-minus_k);

  // 计算 xi 和 zi
  using cache_entry_type = typename cache_accessor<T>::cache_entry_type;
  const cache_entry_type cache = cache_accessor<T>::get_cached_power(-minus_k);

  auto xi = cache_accessor<T>::compute_left_endpoint_for_shorter_interval_case(
      cache, beta_minus_1);
  auto zi = cache_accessor<T>::compute_right_endpoint_for_shorter_interval_case(
      cache, beta_minus_1);

  // 如果左端点不是整数，则增加它
  if (!is_left_endpoint_integer_shorter_interval<T>(exponent)) ++xi;

  // 尝试更大的除数
  ret_value.significand = zi / 10;

  // 如果成功，必要时移除尾随的零并返回
  if (ret_value.significand * 10 >= xi) {
    ret_value.exponent = minus_k + 1;
    ret_value.exponent += remove_trailing_zeros(ret_value.significand);
    return ret_value;
  }

  // 否则，计算 y 的四舍五入
  ret_value.significand =
      cache_accessor<T>::compute_round_up_for_shorter_interval_case(
          cache, beta_minus_1);
  ret_value.exponent = minus_k;

  // 当出现平局时，根据规则选择其中一个
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

template <typename T>
  # Step 1: 整数提升和 Schubfach 乘法器计算。

  # 使用 float_info<T> 中定义的 carrier_uint 类型
  using carrier_uint = typename float_info<T>::carrier_uint;
  # 使用 cache_accessor<T> 中定义的 cache_entry_type 类型
  using cache_entry_type = typename cache_accessor<T>::cache_entry_type;
  # 将输入参数 x 转换为 carrier_uint 类型
  auto br = bit_cast<carrier_uint>(x);

  # 提取尾数位和指数位
  const carrier_uint significand_mask =
      (static_cast<carrier_uint>(1) << float_info<T>::significand_bits) - 1;
  carrier_uint significand = (br & significand_mask);
  const carrier_uint exponent_mask =
      ((static_cast<carrier_uint>(1) << float_info<T>::exponent_bits) - 1)
      << float_info<T>::significand_bits;
  # 计算指数
  int exponent =
      static_cast<int>((br & exponent_mask) >> float_info<T>::significand_bits);

  if (exponent != 0) {  # 检查是否为正常数
    exponent += float_info<T>::exponent_bias - float_info<T>::significand_bits;

    # 较短区间情况；类似于 Schubfach 处理
    if (significand == 0) return shorter_interval_case<T>(exponent);

    significand |=
        (static_cast<carrier_uint>(1) << float_info<T>::significand_bits);
  } else {
    # 亚正常情况；区间始终是规则的
    if (significand == 0) return {0, 0};
  // 计算指数，使用浮点类型T的最小指数减去T的有效位数
  exponent = float_info<T>::min_exponent - float_info<T>::significand_bits;

  // 确定是否包含左端点，如果尾数为偶数则包含
  const bool include_left_endpoint = (significand % 2 == 0);
  // 确定是否包含右端点，与左端点相同
  const bool include_right_endpoint = include_left_endpoint;

  // 计算k和beta
  // 计算负k的值，使用floor_log10_pow2函数计算
  const int minus_k = floor_log10_pow2(exponent) - float_info<T>::kappa;
  // 从缓存中获取已缓存的幂
  const cache_entry_type cache = cache_accessor<T>::get_cached_power(-minus_k);
  // 计算beta-1的值
  const int beta_minus_1 = exponent + floor_log2_pow10(-minus_k);

  // 计算zi和deltai
  // 计算deltai的值，使用compute_delta函数计算
  const uint32_t deltai = cache_accessor<T>::compute_delta(cache, beta_minus_1);
  // 计算zi的值，使用compute_mul函数计算
  const carrier_uint two_fc = significand << 1;
  const carrier_uint two_fr = two_fc | 1;
  const carrier_uint zi = cache_accessor<T>::compute_mul(two_fr << beta_minus_1, cache);

  // 步骤2：尝试更大的除数；如果需要，去除尾部的零

  // 使用zi的上界，可能能够优化除法
  // 计算zi / big_divisor的值
  decimal_fp<T> ret_value;
  ret_value.significand = divide_by_10_to_kappa_plus_1(zi);
  // 计算r的值
  uint32_t r = static_cast<uint32_t>(zi - float_info<T>::big_divisor * ret_value.significand);

  // 如果r大于deltai，则跳转到small_divisor_case_label
  if (r > deltai) {
    goto small_divisor_case_label;
  } 
  // 如果r小于deltai
  else if (r < deltai) {
    // 如果r等于0且不包含右端点且是端点整数，则减小significand的值，更新r的值，跳转到small_divisor_case_label
    if (r == 0 && !include_right_endpoint && is_endpoint_integer<T>(two_fr, exponent, minus_k)) {
      --ret_value.significand;
      r = float_info<T>::big_divisor;
      goto small_divisor_case_label;
    }
  } 
  // 如果r等于deltai
  else {
    // r等于deltai；比较小数部分
    // 检查条件的顺序与论文不同，以利用短路计算
    const carrier_uint two_fl = two_fc - 1;
    // 如果不包含左端点或不是端点整数，或者compute_mul_parity返回false，则跳转到small_divisor_case_label
    if ((!include_left_endpoint || !is_endpoint_integer<T>(two_fl, exponent, minus_k)) && !cache_accessor<T>::compute_mul_parity(two_fl, cache, beta_minus_1)) {
      goto small_divisor_case_label;
    }
  }
    }
  }
  // 设置返回值的指数部分为减去 k 再加上浮点数类型 T 的 kappa 常量再加 1
  ret_value.exponent = minus_k + float_info<T>::kappa + 1;

  // 可能需要移除尾部的零
  ret_value.exponent += remove_trailing_zeros(ret_value.significand);
  // 返回结果值
  return ret_value;

  // 步骤 3: 找到具有较小除数的有效数字
# 定义标签 small_divisor_case_label
small_divisor_case_label:
  # 将 ret_value.significand 乘以 10
  ret_value.significand *= 10;
  # 将 ret_value.exponent 设置为 minus_k + float_info<T>::kappa
  ret_value.exponent = minus_k + float_info<T>::kappa;

  # 定义 mask 为 2^kappa - 1
  const uint32_t mask = (1u << float_info<T>::kappa) - 1;
  # 计算 dist 的值
  auto dist = r - (deltai / 2) + (float_info<T>::small_divisor / 2);

  # 判断 dist 是否能被 2^kappa 整除
  if ((dist & mask) == 0) {
    # 计算 approx_y_parity 的值
    const bool approx_y_parity =
        ((dist ^ (float_info<T>::small_divisor / 2)) & 1) != 0;
    # 将 dist 右移 kappa 位
    dist >>= float_info<T>::kappa;

    # 判断 dist 是否能被 5^kappa 整除
    if (check_divisibility_and_divide_by_pow5<float_info<T>::kappa>(dist)) {
      # 将 ret_value.significand 加上 dist
      ret_value.significand += dist;

      # 检查 z^(f) 是否大于等于 epsilon^(f)
      # 如果 z^(f) >= epsilon^(f)，则可能存在 tie
      if (cache_accessor<T>::compute_mul_parity(two_fc, cache, beta_minus_1) !=
          approx_y_parity) {
        # 如果 z^(f) >= epsilon^(f)，则将 ret_value.significand 减 1
        --ret_value.significand;
      } else {
        # 如果 z^(f) >= epsilon^(f)，则可能存在 tie
        if (is_center_integer<T>(two_fc, exponent, minus_k)) {
          # 如果 y 是整数，则根据奇偶性调整 ret_value.significand
          ret_value.significand = ret_value.significand % 2 == 0
                                      ? ret_value.significand
                                      : ret_value.significand - 1;
        }
      }
    }
    # 如果 dist 不能被 5^kappa 整除，则将 ret_value.significand 加上 dist
    else {
      ret_value.significand += dist;
    }
  }
  # 如果 dist 不能被 2^kappa 整除
  else {
    # 优化 dist / small_divisor 的除法计算
    ret_value.significand +=
        small_division_by_pow10<float_info<T>::kappa>(dist);
  }
  # 返回 ret_value
  return ret_value;
}
}  // namespace dragonbox
// 使用 Steele & White 的 Fixed-Precision Positive Floating-Point Printout ((FPP)^2) 算法对值进行格式化：https://fmt.dev/p372-steele.pdf。
template <typename Double>
void fallback_format(Double d, int num_digits, bool binary32, buffer<char>& buf,
                     int& exp10) {
  bigint numerator;    // (FPP)^2 算法中的 2 * R。
  bigint denominator;  // (FPP)^2 算法中的 2 * S。
  bigint lower;             // (FPP)^2 算法中的 (M^-)。
  bigint upper_store;       // 如果与 lower 不同，则存储 upper 的值。
  bigint* upper = nullptr;  // (FPP)^2 算法中的 (M^+)。
  fp value;
  // 如果 lower 边界更接近，则将分子和分母左移一个或两个额外的位（如果 lower 边界更接近），使 lower 和 upper 成为整数。这样可以在后续计算中消除乘以 2。
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
  // 不变量：value == (numerator / denominator) * pow(10, exp10)。
  if (num_digits < 0) {
    // 生成最短表示形式。
    if (!upper) upper = &lower;  // 如果 upper 为空，则将其指向 lower
    bool even = (value.f & 1) == 0;  // 判断 value.f 是否为偶数
    num_digits = 0;  // 初始化数字位数为 0
    char* data = buf.data();  // 获取 buf 的数据指针
    for (;;) {  // 无限循环
      int digit = numerator.divmod_assign(denominator);  // 使用 numerator 除以 denominator 并将商赋值给 digit
      bool low = compare(numerator, lower) - even < 0;  // 判断 numerator 是否小于等于 lower
      // 判断 numerator + upper 是否大于等于 pow10:
      bool high = add_compare(numerator, *upper, denominator) + even > 0;
      data[num_digits++] = static_cast<char>('0' + digit);  // 将 digit 转换为字符并存入 data 中，然后 num_digits 自增
      if (low || high) {  // 如果 low 或 high 为真
        if (!low) {  // 如果 low 为假
          ++data[num_digits - 1];  // 将 data[num_digits - 1] 的值加一
        } else if (high) {  // 如果 high 为真
          int result = add_compare(numerator, numerator, denominator);  // 比较 numerator 和 denominator 的大小
          // 四舍五入到偶数。
          if (result > 0 || (result == 0 && (digit % 2) != 0))  // 如果 result 大于 0 或者等于 0 且 digit 为奇数
            ++data[num_digits - 1];  // 将 data[num_digits - 1] 的值加一
        }
        buf.try_resize(to_unsigned(num_digits));  // 尝试调整 buf 的大小为 num_digits
        exp10 -= num_digits - 1;  // 更新 exp10
        return;  // 返回
      }
      numerator *= 10;  // numerator 乘以 10
      lower *= 10;  // lower 乘以 10
      if (upper != &lower) *upper *= 10;  // 如果 upper 不等于 lower，则将 upper 乘以 10
    }
  }
  // 生成给定位数的数字。
  exp10 -= num_digits - 1;  // 更新 exp10
  if (num_digits == 0) {  // 如果 num_digits 为 0
    buf.try_resize(1);  // 尝试调整 buf 的大小为 1
    denominator *= 10;  // denominator 乘以 10
    buf[0] = add_compare(numerator, numerator, denominator) > 0 ? '1' : '0';  // 根据比较结果给 buf[0] 赋值
    return;  // 返回
  }
  buf.try_resize(to_unsigned(num_digits));  // 尝试调整 buf 的大小为 num_digits
  for (int i = 0; i < num_digits - 1; ++i) {  // 循环 num_digits - 1 次
    int digit = numerator.divmod_assign(denominator);  // 使用 numerator 除以 denominator 并将商赋值给 digit
    buf[i] = static_cast<char>('0' + digit);  // 将 digit 转换为字符并存入 buf[i] 中
    numerator *= 10;  // numerator 乘以 10
  }
  int digit = numerator.divmod_assign(denominator);  // 使用 numerator 除以 denominator 并将商赋值给 digit
  auto result = add_compare(numerator, numerator, denominator);  // 比较 numerator 和 denominator 的大小
  if (result > 0 || (result == 0 && (digit % 2) != 0)) {  // 如果 result 大于 0 或者等于 0 且 digit 为奇数
    if (digit == 9) {  // 如果 digit 为 9
      const auto overflow = '0' + 10;  // 定义 overflow 为 '0' + 10
      buf[num_digits - 1] = overflow;  // 将 buf[num_digits - 1] 的值设为 overflow
      // 传递进位。
      for (int i = num_digits - 1; i > 0 && buf[i] == overflow; --i) {  // 循环直到 i 为 0 或者 buf[i] 不等于 overflow
        buf[i] = '0';  // 将 buf[i] 的值设为 '0'
        ++buf[i - 1];  // 将 buf[i - 1] 的值加一
      }
      if (buf[0] == overflow) {  // 如果 buf[0] 等于 overflow
        buf[0] = '1';  // 将 buf[0] 的值设为 '1'
        ++exp10;  // exp10 加一
      }
      return;  // 返回
    }
    # 将变量 digit 的值加一
    ++digit;
  # 将 buf 数组中索引为 num_digits - 1 的位置赋值为字符型的 digit 值
  buf[num_digits - 1] = static_cast<char>('0' + digit);
  // 模板函数，用于格式化浮点数为字符串
template <typename T>
int format_float(T value, int precision, float_specs specs, buffer<char>& buf) {
  // 静态断言，确保模板类型不是 float
  static_assert(!std::is_same<T, float>::value, "");
  // 断言 value 大于等于 0
  FMT_ASSERT(value >= 0, "value is negative");

  // 根据格式选择固定点还是浮点数格式
  const bool fixed = specs.format == float_format::fixed;
  // 如果 value 小于等于 0
  if (value <= 0) {  // <= instead of == to silence a warning.
    // 如果 precision 小于等于 0 或者不是固定点格式，则将 '0' 加入到 buf 中并返回 0
    if (precision <= 0 || !fixed) {
      buf.push_back('0');
      return 0;
    }
    // 尝试调整 buf 的大小为 precision，并填充 '0'
    buf.try_resize(to_unsigned(precision));
    std::uninitialized_fill_n(buf.data(), precision, '0');
    return -precision;
  }

  // 如果不使用 grisu 算法，则调用 snprintf_float 函数进行格式化
  if (!specs.use_grisu) return snprintf_float(value, precision, specs, buf);

  // 如果 precision 小于 0
  if (precision < 0) {
    // 使用 Dragonbox 算法进行最短格式的转换
    if (specs.binary32) {
      auto dec = dragonbox::to_decimal(static_cast<float>(value));
      write<char>(buffer_appender<char>(buf), dec.significand);
      return dec.exponent;
    }
    auto dec = dragonbox::to_decimal(static_cast<double>(value));
    write<char>(buffer_appender<char>(buf), dec.significand);
    return dec.exponent;
  }

  // 使用 Grisu + Dragon4 算法进行给定精度的转换
  int exp = 0;
  const int min_exp = -60;  // alpha in Grisu.
  int cached_exp10 = 0;     // K in Grisu.
  fp normalized = normalize(fp(value));
  const auto cached_pow = get_cached_power(
      min_exp - (normalized.e + fp::significand_size), cached_exp10);
  normalized = normalized * cached_pow;
  // 限制精度为 IEEE754 double 中可能的最大有效数字，因为我们不需要生成零
  const int max_double_digits = 767;
  if (precision > max_double_digits) precision = max_double_digits;
  fixed_handler handler{buf.data(), 0, precision, -cached_exp10, fixed};
  // 如果 grisu_gen_digits 返回错误，则进行回退格式化
  if (grisu_gen_digits(normalized, 1, exp, handler) == digits::error) {
    exp += handler.size - cached_exp10 - 1;
    fallback_format(value, handler.precision, specs.binary32, buf, exp);
  } else {
    // 将exp变量增加handler.exp10的值
    exp += handler.exp10;
    // 尝试调整buf的大小为handler.size
    buf.try_resize(to_unsigned(handler.size));
  }
  // 如果不是固定格式且不需要显示小数点
  if (!fixed && !specs.showpoint) {
    // 移除尾部的零
    auto num_digits = buf.size();
    while (num_digits > 0 && buf[num_digits - 1] == '0') {
      --num_digits;
      ++exp;
    }
    // 尝试调整buf的大小为num_digits
    buf.try_resize(num_digits);
  }
  // 返回exp变量的值
  return exp;
}  // namespace detail

template <typename T>
int snprintf_float(T value, int precision, float_specs specs,
                   buffer<char>& buf) {
  // 确保缓冲区容量大于当前大小，否则 MSVC 的 vsnprintf_s 会失败
  FMT_ASSERT(buf.capacity() > buf.size(), "empty buffer");
  static_assert(!std::is_same<T, float>::value, "");

  // 减去 1 以考虑精度的差异，因为我们对一般格式和指数格式都使用 %e
  if (specs.format == float_format::general ||
      specs.format == float_format::exp)
    precision = (precision >= 0 ? precision : 6) - 1;

  // 构建格式字符串
  enum { max_format_size = 7 };  // 最长的格式为 "%#.*Le"
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

  // 使用 snprintf 进行格式化
  auto offset = buf.size();
  for (;;) {
    auto begin = buf.data() + offset;
    auto capacity = buf.capacity() - offset;
#ifdef FMT_FUZZ
    if (precision > 100000)
      throw std::runtime_error(
          "fuzz mode - avoid large allocation inside snprintf");
#endif
    // 抑制关于非文字格式字符串的警告
    // 由于 MinGW 中的一个 bug，不能使用 auto (#1532)
    int (*snprintf_ptr)(char*, size_t, const char*, ...) = FMT_SNPRINTF;
    int result = precision >= 0
                     ? snprintf_ptr(begin, capacity, format, precision, value)
                     : snprintf_ptr(begin, capacity, format, value);
    if (result < 0) {
      // 缓冲区将呈指数增长
      buf.try_reserve(buf.capacity() + 1);
      continue;
    // 将结果转换为无符号整数
    auto size = to_unsigned(result);
    // 如果大小等于容量，意味着最后一个字符被截断
    if (size >= capacity) {
      buf.try_reserve(size + offset + 1);  // 为终止符'\0'添加1
      continue;
    }
    // 定义一个判断字符是否为数字的lambda函数
    auto is_digit = [](char c) { return c >= '0' && c <= '9'; };
    // 如果格式为固定浮点数
    if (specs.format == float_format::fixed) {
      if (precision == 0) {
        buf.try_resize(size);
        return 0;
      }
      // 查找并移除小数点
      auto end = begin + size, p = end;
      do {
        --p;
      } while (is_digit(*p));
      int fraction_size = static_cast<int>(end - p - 1);
      std::memmove(p, p + 1, to_unsigned(fraction_size));
      buf.try_resize(size - 1);
      return -fraction_size;
    }
    // 如果格式为十六进制浮点数
    if (specs.format == float_format::hex) {
      buf.try_resize(size + offset);
      return 0;
    }
    // 查找并解析指数部分
    auto end = begin + size, exp_pos = end;
    do {
      --exp_pos;
    } while (*exp_pos != 'e');
    char sign = exp_pos[1];
    assert(sign == '+' || sign == '-');
    int exp = 0;
    auto p = exp_pos + 2;  // 跳过'e'和符号
    do {
      assert(is_digit(*p));
      exp = exp * 10 + (*p++ - '0');
    } while (p != end);
    if (sign == '-') exp = -exp;
    int fraction_size = 0;
    if (exp_pos != begin + 1) {
      // 移除尾部的零
      auto fraction_end = exp_pos - 1;
      while (*fraction_end == '0') --fraction_end;
      // 将小数部分左移以去除小数点
      fraction_size = static_cast<int>(fraction_end - begin - 1);
      std::memmove(begin + 1, begin + 2, to_unsigned(fraction_size));
    }
    buf.try_resize(to_unsigned(fraction_size) + offset + 1);
    return exp - fraction_size;
  }
// 从缓冲区中解码下一个字符c，如果出现错误则在e中报告
// 由于这是一个无分支解码器，无论实际的下一个字符的长度如何，都将从缓冲区中读取四个字节。这意味着缓冲区必须至少有三个字节的零填充，跟随数据流的结束。
// 错误将在e中报告，如果解析的字符出现无效：无效的字节序列，非规范的编码，或者是代理半个。
// 函数返回指向下一个字符的指针。当发生错误时，这个指针将是一个依赖于特定错误的猜测，但它将至少前进一个字节。
// 解码 UTF-8 编码的字符，返回下一个字符的指针
inline const char* utf8_decode(const char* buf, uint32_t* c, int* e) {
  // UTF-8 编码长度表
  static const char lengths[] = {1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1,
                                 1, 1, 1, 1, 1, 0, 0, 0, 0, 0, 0,
                                 0, 0, 2, 2, 2, 2, 3, 3, 4, 0};
  // UTF-8 编码掩码
  static const int masks[] = {0x00, 0x7f, 0x1f, 0x0f, 0x07};
  // UTF-8 编码最小值
  static const uint32_t mins[] = {4194304, 0, 128, 2048, 65536};
  // 移位常量
  static const int shiftc[] = {0, 18, 12, 6, 0};
  static const int shifte[] = {0, 6, 4, 2, 0};

  // 将 buf 转换为无符号字符指针
  auto s = reinterpret_cast<const unsigned char*>(buf);
  // 计算字符长度
  int len = lengths[s[0] >> 3];

  // 提前计算下一个字符的指针，以便下一次迭代可以开始处理下一个字符
  const char* next = buf + len + !len;

  // 假设一个四字节字符并加载四个字节，未使用的位被移出
  *c = uint32_t(s[0] & masks[len]) << 18;
  *c |= uint32_t(s[1] & 0x3f) << 12;
  *c |= uint32_t(s[2] & 0x3f) << 6;
  *c |= uint32_t(s[3] & 0x3f) << 0;
  *c >>= shiftc[len];

  // 累积各种错误条件
  *e = (*c < mins[len]) << 6;       // 非规范编码
  *e |= ((*c >> 11) == 0x1b) << 7;  // 代理对半？
  *e |= (*c > 0x10FFFF) << 8;       // 超出范围？
  *e |= (s[1] & 0xc0) >> 2;
  *e |= (s[2] & 0xc0) >> 4;
  *e |= (s[3]) >> 6;
  *e ^= 0x2a;  // 每个尾字节的前两位是否正确？
  *e >>= shifte[len];

  return next;
}

// 字符串化器结构体
struct stringifier {
  // 重载函数调用操作符，用于将值转换为字符串
  template <typename T> FMT_INLINE std::string operator()(T value) const {
    return to_string(value);
  }
  // 重载函数调用操作符，用于处理格式化参数
  std::string operator()(basic_format_arg<format_context>::handle h) const {
    // 创建内存缓冲区
    memory_buffer buf;
    // 创建格式解析上下文和格式化上下文
    format_parse_context parse_ctx({});
    format_context format_ctx(buffer_appender<char>(buf), {}, {});
    // 格式化参数
    h.format(parse_ctx, format_ctx);
    // 将格式化结果转换为字符串
    return to_string(buf);
  }
};
}  // namespace detail
// 定义一个模板特化，用于格式化 detail::bigint 类型的数据
template <> struct formatter<detail::bigint> {
  // 解析函数，返回格式化解析上下文的迭代器
  format_parse_context::iterator parse(format_parse_context& ctx) {
    return ctx.begin();
  }

  // 格式化函数，将 detail::bigint 类型的数据格式化为字符串
  format_context::iterator format(const detail::bigint& n,
                                  format_context& ctx) {
    auto out = ctx.out();
    bool first = true;
    // 遍历 bigint 类型的数据的每个位
    for (auto i = n.bigits_.size(); i > 0; --i) {
      auto value = n.bigits_[i - 1u];
      // 格式化每个位的值为十六进制
      if (first) {
        out = format_to(out, "{:x}", value);
        first = false;
        continue;
      }
      out = format_to(out, "{:08x}", value);
    }
    // 如果指数大于 0，则格式化为指数形式
    if (n.exp_ > 0)
      out = format_to(out, "p{}", n.exp_ * detail::bigint::bigit_bits);
    return out;
  }
};

// 定义 utf8_to_utf16 类的构造函数
FMT_FUNC detail::utf8_to_utf16::utf8_to_utf16(string_view s) {
  // 定义转码函数
  auto transcode = [this](const char* p) {
    auto cp = uint32_t();
    auto error = 0;
    // 解码 utf8 字符串为 unicode 码点
    p = utf8_decode(p, &cp, &error);
    if (error != 0) FMT_THROW(std::runtime_error("invalid utf8"));
    // 将 unicode 码点转换为 utf16 编码，并存入 buffer_
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
  // 如果字符串长度大于等于 block_size，则按块转码
  if (s.size() >= block_size) {
    for (auto end = p + s.size() - block_size + 1; p < end;) p = transcode(p);
  }
  // 处理剩余的字符
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

// 格式化系统错误信息
FMT_FUNC void format_system_error(detail::buffer<char>& out, int error_code,
                                  string_view message) FMT_NOEXCEPT {
  FMT_TRY {
    memory_buffer buf;
    buf.resize(inline_buffer_size);
    // ...
  }
    // 无限循环，直到条件被打破
    for (;;) {
      // 创建指向缓冲区的指针
      char* system_message = &buf[0];
      // 调用safe_strerror函数，将错误码转换为系统错误消息，将结果存储在system_message中
      int result =
          detail::safe_strerror(error_code, system_message, buf.size());
      // 如果结果为0，表示成功获取系统错误消息，将消息格式化并写入输出流，然后返回
      if (result == 0) {
        format_to(detail::buffer_appender<char>(out), "{}: {}", message,
                  system_message);
        return;
      }
      // 如果结果不是ERANGE，表示无法获取错误消息，跳出循环，报告错误码
      if (result != ERANGE)
        break;  // Can't get error message, report error code instead.
      // 调整缓冲区大小为原来的两倍
      buf.resize(buf.size() * 2);
    }
  }
  // 捕获任何异常
  FMT_CATCH(...) {}
  // 格式化错误码并写入输出流
  format_error_code(out, error_code, message);
# 定义错误处理器的错误处理函数，当出现错误时抛出格式化错误
FMT_FUNC void detail::error_handler::on_error(const char* message) {
  FMT_THROW(format_error(message));
}

# 报告系统错误，传入错误码和消息
FMT_FUNC void report_system_error(int error_code,
                                  fmt::string_view message) FMT_NOEXCEPT {
  # 调用 report_error 函数报告系统错误
  report_error(format_system_error, error_code, message);
}

# 格式化字符串和参数，返回格式化后的字符串
FMT_FUNC std::string detail::vformat(string_view format_str, format_args args) {
  # 如果格式化字符串长度为2且内容为"{}"，则处理第一个参数并返回
  if (format_str.size() == 2 && equal2(format_str.data(), "{}")) {
    auto arg = args.get(0);
    if (!arg) error_handler().on_error("argument not found");
    return visit_format_arg(stringifier(), arg);
  }
  # 创建内存缓冲区
  memory_buffer buffer;
  # 将格式化后的字符串和参数写入缓冲区
  detail::vformat_to(buffer, format_str, args);
  # 将缓冲区内容转换为字符串并返回
  return to_string(buffer);
}

# 如果是 Windows 平台，则定义 detail 命名空间和 WriteConsoleW 函数
#ifdef _WIN32
namespace detail {
using dword = conditional_t<sizeof(long) == 4, unsigned long, unsigned>;
extern "C" __declspec(dllimport) int __stdcall WriteConsoleW(  //
    void*, const void*, dword, dword*, void*);
}  // namespace detail
#endif

# 打印格式化后的字符串到文件流
FMT_FUNC void vprint(std::FILE* f, string_view format_str, format_args args) {
  # 创建内存缓冲区
  memory_buffer buffer;
  # 将格式化后的字符串和参数写入缓冲区
  detail::vformat_to(buffer, format_str,
                     basic_format_args<buffer_context<char>>(args));
  # 如果是 Windows 平台
  # 判断文件流是否是控制台，如果是则将内容写入控制台
  # 否则将内容写入文件流
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
#endif
  # 将缓冲区内容写入文件流
  detail::fwrite_fully(buffer.data(), 1, buffer.size(), f);
}

# 如果是 Windows 平台，则定义打印函数，假设使用传统编码
#ifdef _WIN32
// Print assuming legacy (non-Unicode) encoding.
// 定义一个函数，用于在文件流中打印乱码字符串
FMT_FUNC void detail::vprint_mojibake(std::FILE* f, string_view format_str,
                                      format_args args) {
  // 创建一个内存缓冲区
  memory_buffer buffer;
  // 将格式化后的字符串写入到内存缓冲区中
  detail::vformat_to(buffer, format_str,
                     basic_format_args<buffer_context<char>>(args));
  // 将内存缓冲区的数据完全写入到文件流中
  fwrite_fully(buffer.data(), 1, buffer.size(), f);
}
// 结束定义 detail 命名空间

// 定义一个函数，用于在标准输出流中打印格式化字符串
FMT_FUNC void vprint(string_view format_str, format_args args) {
  // 调用 vprint_mojibake 函数，将格式化后的字符串打印到标准输出流中
  vprint(stdout, format_str, args);
}

// 结束定义 fmt 命名空间
FMT_END_NAMESPACE

// 结束 if 语句
#endif  // FMT_FORMAT_INL_H_
```