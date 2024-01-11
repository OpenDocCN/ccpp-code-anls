# `xmrig\src\3rdparty\fmt\format.h`

```
/*
 C++的格式化库
 版权所有（c）2012 - 现在，Victor Zverovich

 在遵守以下条件的情况下，特此免费授予任何获得本软件及相关文档文件（以下简称“软件”）副本的人无限制地处理本软件的权限，包括但不限于使用、复制、修改、合并、发布、分发、再许可和/或出售本软件的副本，并允许获得本软件的人员这样做，但须遵守以下条件：

 上述版权声明和本许可声明应包含在所有副本或重要部分的软件中。

 本软件按“原样”提供，不提供任何形式的明示或暗示的担保，包括但不限于对特定目的的适销性、适用性和非侵权性的担保。在任何情况下，作者或版权持有人均不对任何索赔、损害或其他责任负责，无论是在合同行为、侵权行为还是其他方面，由于软件或使用或其他方式与软件的使用有关而产生的、引起的或与之相关的。

 --- 许可证的可选例外 ---

 作为例外，如果由于编译源代码的结果，此软件的部分内容被嵌入到这种源代码的机器可执行对象形式中，您可以在不包括上述版权和许可声明的情况下重新分发这些嵌入的部分。

 */

#ifndef FMT_FORMAT_H_
#define FMT_FORMAT_H_

#include <algorithm>  // 包含算法库
#include <cerrno>  // 包含错误码库
#include <cmath>  // 包含数学库
#include <cstdint>  // 包含整数类型库
#include <limits>  // 包含数值极限库
#include <memory>  // 包含内存管理库
#include <stdexcept>  // 包含标准异常库

#include "core.h"  // 包含核心库

#ifdef __INTEL_COMPILER  // 如果是英特尔编译器
#  define FMT_ICC_VERSION __INTEL_COMPILER  // 定义FMT_ICC_VERSION为__INTEL_COMPILER
#elif defined(__ICL)  // 如果是ICL编译器
#  define FMT_ICC_VERSION __ICL  // 定义FMT_ICC_VERSION为__ICL
#else  // 否则
#  define FMT_ICC_VERSION 0  // 定义FMT_ICC_VERSION为0
#endif

#ifdef __NVCC__  // 如果是NVCC编译器
#  define FMT_CUDA_VERSION (__CUDACC_VER_MAJOR__ * 100 + __CUDACC_VER_MINOR__)  // 定义FMT_CUDA_VERSION为__CUDACC_VER_MAJOR__和__CUDACC_VER_MINOR__的乘积
#else  // 否则
#  define FMT_CUDA_VERSION 0  // 定义FMT_CUDA_VERSION为0
#endif

#ifdef __has_builtin  // 如果有内建函数
#ifndef FMT_USE_USER_DEFINED_LITERALS
// 如果未定义 FMT_USE_USER_DEFINED_LITERALS，则执行以下操作
// EDG based compilers (Intel, NVIDIA, Elbrus, etc), GCC and MSVC support UDLs.
// EDG 编译器（Intel、NVIDIA、Elbrus 等）、GCC 和 MSVC 支持用户定义字面值
#ifndef FMT_USE_UDL_TEMPLATE
// 如果未定义 FMT_USE_UDL_TEMPLATE，则进行以下判断
// 对于使用用户自定义字面值的情况，以及编译器版本和特性的判断，确定是否使用用户自定义字面值
#  if FMT_USE_USER_DEFINED_LITERALS &&                         \
      (!defined(__EDG_VERSION__) || __EDG_VERSION__ >= 501) && \
      ((FMT_GCC_VERSION >= 604 && __cplusplus >= 201402L) ||   \
       FMT_CLANG_VERSION >= 304) &&                            \
      !defined(__PGI) && !defined(__NVCC__)
// 如果满足条件，则定义 FMT_USE_UDL_TEMPLATE 为 1
#    define FMT_USE_UDL_TEMPLATE 1
#  else
// 否则定义 FMT_USE_UDL_TEMPLATE 为 0
#    define FMT_USE_UDL_TEMPLATE 0
#  endif
#endif

#ifndef FMT_USE_FLOAT
// 如果未定义 FMT_USE_FLOAT，则定义为 1
#  define FMT_USE_FLOAT 1
#endif

#ifndef FMT_USE_DOUBLE
// 如果未定义 FMT_USE_DOUBLE，则定义为 1
#  define FMT_USE_DOUBLE 1
#endif

#ifndef FMT_USE_LONG_DOUBLE
// 如果未定义 FMT_USE_LONG_DOUBLE，则定义为 1
#  define FMT_USE_LONG_DOUBLE 1
#endif

// 如果未定义 FMT_REDUCE_INT_INSTANTIATIONS，则定义为 0
#if !defined(FMT_REDUCE_INT_INSTANTIATIONS)
#  define FMT_REDUCE_INT_INSTANTIATIONS 0
#endif

// 如果满足条件，则定义 FMT_BUILTIN_CLZ(n) 为 __builtin_clz(n)
#if (FMT_GCC_VERSION || FMT_HAS_BUILTIN(__builtin_clz)) && !FMT_MSC_VER
#  define FMT_BUILTIN_CLZ(n) __builtin_clz(n)
#endif
// 如果满足条件，则定义 FMT_BUILTIN_CLZLL(n) 为 __builtin_clzll(n)
#if (FMT_GCC_VERSION || FMT_HAS_BUILTIN(__builtin_clzll)) && !FMT_MSC_VER
#  define FMT_BUILTIN_CLZLL(n) __builtin_clzll(n)
#endif
// 如果满足条件，则定义 FMT_BUILTIN_CTZ(n) 为 __builtin_ctz(n)
#if (FMT_GCC_VERSION || FMT_HAS_BUILTIN(__builtin_ctz))
#  define FMT_BUILTIN_CTZ(n) __builtin_ctz(n)
#endif
// 如果满足条件，则定义 FMT_BUILTIN_CTZLL(n) 为 __builtin_ctzll(n)
#if (FMT_GCC_VERSION || FMT_HAS_BUILTIN(__builtin_ctzll))
// 定义 FMT_BUILTIN_CTZLL 宏，使用 __builtin_ctzll(n) 内建函数
#  define FMT_BUILTIN_CTZLL(n) __builtin_ctzll(n)
#endif

// 如果是 MSC 编译器，包含 <intrin.h> 头文件，用于 _BitScanReverse[64], _BitScanForward[64], _umul128
#if FMT_MSC_VER
#  include <intrin.h>  // _BitScanReverse[64], _BitScanForward[64], _umul128
#endif

// 一些编译器伪装成 MSVC 和 GCC-likes，或者支持 __builtin_clz 和 __builtin_clzll，所以只有在 clz 和 clzll 内建函数不可用时，才使用 MSVC 内建函数定义 FMT_BUILTIN_CLZ
#if FMT_MSC_VER && !defined(FMT_BUILTIN_CLZLL) && \
    !defined(FMT_BUILTIN_CTZLL) && !defined(_MANAGED)
FMT_BEGIN_NAMESPACE
namespace detail {
// 避免 Clang with Microsoft CodeGen 的 -Wunknown-pragmas 警告
#  ifndef __clang__
#    pragma intrinsic(_BitScanForward)
#    pragma intrinsic(_BitScanReverse)
#  endif
#  if defined(_WIN64) && !defined(__clang__)
#    pragma intrinsic(_BitScanForward64)
#    pragma intrinsic(_BitScanReverse64)
#  endif

// 定义 clz 函数，计算 32 位整数的前导零
inline int clz(uint32_t x) {
  unsigned long r = 0;
  _BitScanReverse(&r, x);
  FMT_ASSERT(x != 0, "");
  // 静态分析警告使用未初始化数据 "r"，但唯一可能发生的情况是 "x" 为 0，而调用者保证不会发生这种情况
  FMT_SUPPRESS_MSC_WARNING(6102)
  return 31 ^ static_cast<int>(r);
}
#  define FMT_BUILTIN_CLZ(n) detail::clz(n)

// 定义 clzll 函数，计算 64 位整数的前导零
inline int clzll(uint64_t x) {
  unsigned long r = 0;
#  ifdef _WIN64
  _BitScanReverse64(&r, x);
#  else
  // 扫描高 32 位
  if (_BitScanReverse(&r, static_cast<uint32_t>(x >> 32))) return 63 ^ (r + 32);
  // 扫描低 32 位
  _BitScanReverse(&r, static_cast<uint32_t>(x));
#  endif
  FMT_ASSERT(x != 0, "");
  FMT_SUPPRESS_MSC_WARNING(6102)  // 抑制虚假的静态分析警告
  return 63 ^ static_cast<int>(r);
}
#  define FMT_BUILTIN_CLZLL(n) detail::clzll(n)

// 定义 ctz 函数，计算 32 位整数的尾随零
inline int ctz(uint32_t x) {
  unsigned long r = 0;
  _BitScanForward(&r, x);
  FMT_ASSERT(x != 0, "");
  FMT_SUPPRESS_MSC_WARNING(6102)  // 抑制虚假的静态分析警告
  return static_cast<int>(r);
}
#  define FMT_BUILTIN_CTZ(n) detail::ctz(n)
// 计算无符号64位整数 x 的末尾零位的数量
inline int ctzll(uint64_t x) {
  unsigned long r = 0;
  // 断言 x 不为0
  FMT_ASSERT(x != 0, "");
  // 抑制一个虚假的静态分析警告
  FMT_SUPPRESS_MSC_WARNING(6102)  // Suppress a bogus static analysis warning.
#  ifdef _WIN64
  // 在 64 位 Windows 系统上使用内置函数 _BitScanForward64
  _BitScanForward64(&r, x);
#  else
  // 在非 64 位 Windows 系统上，先扫描低 32 位
  if (_BitScanForward(&r, static_cast<uint32_t>(x))) return static_cast<int>(r);
  // 再扫描高 32 位
  _BitScanForward(&r, static_cast<uint32_t>(x >> 32));
  r += 32;
#  endif
  return static_cast<int>(r);
}
#  define FMT_BUILTIN_CTZLL(n) detail::ctzll(n)
}  // namespace detail
FMT_END_NAMESPACE
#endif

// 启用已弃用的数字对齐
#ifndef FMT_DEPRECATED_NUMERIC_ALIGN
#  define FMT_DEPRECATED_NUMERIC_ALIGN 0
#endif

FMT_BEGIN_NAMESPACE
namespace detail {

// 一个等价于 `*reinterpret_cast<Dest*>(&source)` 的函数，避免了未定义行为（例如由于类型别名引起的）
// 示例：uint64_t d = bit_cast<uint64_t>(2.718);
template <typename Dest, typename Source>
inline Dest bit_cast(const Source& source) {
  static_assert(sizeof(Dest) == sizeof(Source), "size mismatch");
  Dest dest;
  std::memcpy(&dest, &source, sizeof(dest));
  return dest;
}

// 判断系统是否为大端序
inline bool is_big_endian() {
  const auto u = 1u;
  struct bytes {
    char data[sizeof(u)];
  };
  return bit_cast<bytes>(u).data[0] == 0;
}

// 一个用于没有 uintptr_t 的系统的 fallback 实现
struct fallback_uintptr {
  unsigned char value[sizeof(void*)];

  fallback_uintptr() = default;
  explicit fallback_uintptr(const void* p) {
    *this = bit_cast<fallback_uintptr>(p);
    if (is_big_endian()) {
      for (size_t i = 0, j = sizeof(void*) - 1; i < j; ++i, --j)
        std::swap(value[i], value[j]);
    }
  }
};
#ifdef UINTPTR_MAX
using uintptr_t = ::uintptr_t;
inline uintptr_t to_uintptr(const void* p) { return bit_cast<uintptr_t>(p); }
#else
using uintptr_t = fallback_uintptr;
inline fallback_uintptr to_uintptr(const void* p) {
  return fallback_uintptr(p);
}
#endif

// 返回类型 T 的最大可能值。与
// 定义一个模板函数，返回指定类型的最大值
template <typename T> constexpr T max_value() {
  return (std::numeric_limits<T>::max)();
}
// 定义一个模板函数，返回指定类型的位数
template <typename T> constexpr int num_bits() {
  return std::numeric_limits<T>::digits;
}
// 对于128位整数，特化模板函数num_bits，返回固定的位数128
template <> constexpr int num_bits<int128_t>() { return 128; }
template <> constexpr int num_bits<uint128_t>() { return 128; }
template <> constexpr int num_bits<fallback_uintptr>() {
  return static_cast<int>(sizeof(void*) *
                          std::numeric_limits<unsigned char>::digits);
}

// 定义一个函数，用于假设条件成立
FMT_INLINE void assume(bool condition) {
  (void)condition;
  // 如果支持__builtin_assume，则使用__builtin_assume假设条件成立
#if FMT_HAS_BUILTIN(__builtin_assume)
  __builtin_assume(condition);
#endif
}

// 用于在C++20之前模拟iterator_t的近似类型
template <typename T>
using iterator_t = decltype(std::begin(std::declval<T&>()));
template <typename T> using sentinel_t = decltype(std::end(std::declval<T&>()));

// 用于解决在C++17之前std::string没有mutable data()的问题
template <typename Char> inline Char* get_data(std::basic_string<Char>& s) {
  return &s[0];
}
template <typename Container>
inline typename Container::value_type* get_data(Container& c) {
  return c.data();
}

// 如果定义了_SECURE_SCL并且为真，则使用checked_array_iterator，否则直接使用指针
template <typename T> using checked_ptr = stdext::checked_array_iterator<T*>;
template <typename T> checked_ptr<T> make_checked(T* p, size_t size) {
  return {p, size};
}
// 如果没有定义_SECURE_SCL或者为假，则直接返回指针
template <typename T> using checked_ptr = T*;
template <typename T> inline T* make_checked(T* p, size_t) { return p; }

// 对于连续存储的容器，定义一个函数返回checked_ptr
template <typename Container, FMT_ENABLE_IF(is_contiguous<Container>::value)>
#if FMT_CLANG_VERSION
__attribute__((no_sanitize("undefined")))
#endif
inline checked_ptr<typename Container::value_type>
// 在容器末尾预留空间，返回指向预留空间的迭代器
template <typename Container>
reserve(std::back_insert_iterator<Container> it, size_t n) {
  // 获取容器引用
  Container& c = get_container(it);
  // 获取容器当前大小
  size_t size = c.size();
  // 调整容器大小以预留空间
  c.resize(size + n);
  // 返回指向预留空间的迭代器
  return make_checked(get_data(c) + size, n);
}

// 在缓冲区末尾预留空间，返回缓冲区迭代器
template <typename T>
inline buffer_appender<T> reserve(buffer_appender<T> it, size_t n) {
  // 获取缓冲区引用
  buffer<T>& buf = get_container(it);
  // 尝试为缓冲区预留空间
  buf.try_reserve(buf.size() + n);
  // 返回缓冲区迭代器
  return it;
}

// 保留迭代器位置，不进行任何操作
template <typename Iterator> inline Iterator& reserve(Iterator& it, size_t) {
  return it;
}

// 将输出迭代器转换为指针，返回空指针
template <typename T, typename OutputIt>
constexpr T* to_pointer(OutputIt, size_t) {
  return nullptr;
}
// 将缓冲区迭代器转换为指针，返回指向预留空间的指针
template <typename T> T* to_pointer(buffer_appender<T> it, size_t n) {
  // 获取缓冲区引用
  buffer<T>& buf = get_container(it);
  // 获取当前缓冲区大小
  auto size = buf.size();
  // 如果缓冲区容量不足，则返回空指针
  if (buf.capacity() < size + n) return nullptr;
  // 调整缓冲区大小以预留空间
  buf.try_resize(size + n);
  // 返回指向预留空间的指针
  return buf.data() + size;
}

// 返回基础迭代器，用于支持连续存储的容器
template <typename Container, FMT_ENABLE_IF(is_contiguous<Container>::value)>
inline std::back_insert_iterator<Container> base_iterator(
    std::back_insert_iterator<Container>& it,
    checked_ptr<typename Container::value_type>) {
  return it;
}

// 返回基础迭代器，用于不支持连续存储的容器
template <typename Iterator>
inline Iterator base_iterator(Iterator, Iterator it) {
  return it;
}

// 计数迭代器，用于计算写入的对象数量并丢弃它们
class counting_iterator {
 private:
  size_t count_;

 public:
  using iterator_category = std::output_iterator_tag;
  using difference_type = std::ptrdiff_t;
  using pointer = void;
  using reference = void;
  using _Unchecked_type = counting_iterator;  // Mark iterator as checked.

  struct value_type {
    template <typename T> void operator=(const T&) {}
  };

  counting_iterator() : count_(0) {}

  // 返回当前写入对象的数量
  size_t count() const { return count_; }

  // 前置递增运算符，增加写入对象的数量
  counting_iterator& operator++() {
    ++count_;
    return *this;
  }
  // 后置递增运算符，增加写入对象的数量
  counting_iterator operator++(int) {
    auto it = *this;
    ++*this;
    return it;
  }

  // 重载加法运算符，增加写入对象的数量
  friend counting_iterator operator+(counting_iterator it, difference_type n) {
    it.count_ += static_cast<size_t>(n);
    # 返回迭代器指向的元素
    return it;
  }

  # 重载解引用操作符，返回一个默认构造的值
  value_type operator*() const { return {}; }
// 定义一个模板类，用于迭代输出，并且可以截断输出并计算写入的对象数量
template <typename OutputIt> class truncating_iterator_base {
 protected:
  OutputIt out_; // 输出迭代器
  size_t limit_; // 截断限制
  size_t count_; // 写入对象数量

  // 构造函数，初始化输出迭代器和截断限制
  truncating_iterator_base(OutputIt out, size_t limit)
      : out_(out), limit_(limit), count_(0) {}

 public:
  using iterator_category = std::output_iterator_tag; // 迭代器类型
  using value_type = typename std::iterator_traits<OutputIt>::value_type; // 值类型
  using difference_type = void; // 差值类型
  using pointer = void; // 指针类型
  using reference = void; // 引用类型
  using _Unchecked_type =
      truncating_iterator_base;  // 标记迭代器为已检查类型

  // 返回基础输出迭代器
  OutputIt base() const { return out_; }
  // 返回写入对象数量
  size_t count() const { return count_; }
};

// 一个输出迭代器，可以截断输出并计算写入的对象数量
template <typename OutputIt,
          typename Enable = typename std::is_void<
              typename std::iterator_traits<OutputIt>::value_type>::type>
class truncating_iterator;

// 针对非 void 类型的模板特化
template <typename OutputIt>
class truncating_iterator<OutputIt, std::false_type>
    : public truncating_iterator_base<OutputIt> {
  mutable typename truncating_iterator_base<OutputIt>::value_type blackhole_; // 用于截断的黑洞对象

 public:
  using value_type = typename truncating_iterator_base<OutputIt>::value_type; // 值类型

  // 构造函数，初始化输出迭代器和截断限制
  truncating_iterator(OutputIt out, size_t limit)
      : truncating_iterator_base<OutputIt>(out, limit) {}

  // 前置递增运算符重载
  truncating_iterator& operator++() {
    if (this->count_++ < this->limit_) ++this->out_;
    return *this;
  }

  // 后置递增运算符重载
  truncating_iterator operator++(int) {
    auto it = *this;
    ++*this;
    return it;
  }

  // 解引用运算符重载
  value_type& operator*() const {
    return this->count_ < this->limit_ ? *this->out_ : blackhole_;
  }
};

// 针对 void 类型的模板特化
template <typename OutputIt>
class truncating_iterator<OutputIt, std::true_type>
    : public truncating_iterator_base<OutputIt> {
 public:
  // 构造函数，初始化输出迭代器和截断限制
  truncating_iterator(OutputIt out, size_t limit)
      : truncating_iterator_base<OutputIt>(out, limit) {}

  // 赋值运算符重载
  template <typename T> truncating_iterator& operator=(T val) {
    # 如果计数小于限制，则将值赋给输出指针所指向的位置，并递增计数
    if (this->count_++ < this->limit_) *this->out_++ = val;
    # 返回当前对象的引用
    return *this;
  }

  # 前置递增运算符重载，返回当前对象的引用
  truncating_iterator& operator++() { return *this; }
  # 后置递增运算符重载，返回当前对象的引用
  truncating_iterator& operator++(int) { return *this; }
  # 解引用运算符重载，返回当前对象的引用
  truncating_iterator& operator*() { return *this; }
};

// 计算字符串中的代码点数量
template <typename Char>
inline size_t count_code_points(basic_string_view<Char> s) {
  return s.size();
}

// 计算 UTF-8 字符串中的代码点数量
inline size_t count_code_points(basic_string_view<char> s) {
  const char* data = s.data();
  size_t num_code_points = 0;
  for (size_t i = 0, size = s.size(); i != size; ++i) {
    if ((data[i] & 0xc0) != 0x80) ++num_code_points;
  }
  return num_code_points;
}

// 计算 char8_type 类型字符串中的代码点数量
inline size_t count_code_points(basic_string_view<char8_type> s) {
  return count_code_points(basic_string_view<char>(
      reinterpret_cast<const char*>(s.data()), s.size()));
}

// 查找字符串中第 n 个代码点的索引
template <typename Char>
inline size_t code_point_index(basic_string_view<Char> s, size_t n) {
  size_t size = s.size();
  return n < size ? n : size;
}

// 在 UTF-8 字符串中计算第 n 个代码点的索引
inline size_t code_point_index(basic_string_view<char8_type> s, size_t n) {
  const char8_type* data = s.data();
  size_t num_code_points = 0;
  for (size_t i = 0, size = s.size(); i != size; ++i) {
    if ((data[i] & 0xc0) != 0x80 && ++num_code_points > n) {
      return i;
    }
  }
  return s.size();
}

// 判断是否需要进行字符转换
template <typename InputIt, typename OutChar>
using needs_conversion = bool_constant<
    std::is_same<typename std::iterator_traits<InputIt>::value_type,
                 char>::value &&
    std::is_same<OutChar, char8_type>::value>;

// 复制字符串，不需要进行字符转换
template <typename OutChar, typename InputIt, typename OutputIt,
          FMT_ENABLE_IF(!needs_conversion<InputIt, OutChar>::value)>
OutputIt copy_str(InputIt begin, InputIt end, OutputIt it) {
  return std::copy(begin, end, it);
}

// 复制字符串，需要进行字符转换
template <typename OutChar, typename InputIt, typename OutputIt,
          FMT_ENABLE_IF(needs_conversion<InputIt, OutChar>::value)>
OutputIt copy_str(InputIt begin, InputIt end, OutputIt it) {
  return std::transform(begin, end, it,
                        [](char c) { return static_cast<char8_type>(c); });
}

// 字符串转换模板
template <typename Char, typename InputIt>
// 定义一个内联函数，用于将字符串从输入迭代器范围复制到计数迭代器指定的位置
inline counting_iterator copy_str(InputIt begin, InputIt end,
                                  counting_iterator it) {
  return it + (end - begin);
}

// 定义一个模板别名，用于检查是否 T 类型是快速浮点数类型
template <typename T>
using is_fast_float = bool_constant<std::numeric_limits<T>::is_iec559 &&
                                    sizeof(T) <= sizeof(double)>;

// 如果未定义 FMT_USE_FULL_CACHE_DRAGONBOX，则定义为 0
#ifndef FMT_USE_FULL_CACHE_DRAGONBOX
#  define FMT_USE_FULL_CACHE_DRAGONBOX 0
#endif

// 定义 buffer 类模板的成员函数 append，用于将指定范围的数据追加到 buffer 对象中
template <typename T>
template <typename U>
void buffer<T>::append(const U* begin, const U* end) {
  do {
    auto count = to_unsigned(end - begin);
    // 尝试保留足够的空间以容纳追加的数据
    try_reserve(size_ + count);
    auto free_cap = capacity_ - size_;
    // 如果剩余容量小于要追加的数据量，则只追加剩余容量大小的数据
    if (free_cap < count) count = free_cap;
    // 将指定范围的数据复制到 buffer 对象的末尾
    std::uninitialized_copy_n(begin, count, make_checked(ptr_ + size_, count));
    size_ += count;
    begin += count;
  } while (begin != end);
}

// 定义 iterator_buffer 类模板的成员函数 flush，用于将数据从 buffer 中复制到输出迭代器中，并清空 buffer
template <typename OutputIt, typename T, typename Traits>
void iterator_buffer<OutputIt, T, Traits>::flush() {
  out_ = std::copy_n(data_, this->limit(this->size()), out_);
  this->clear();
}
}  // namespace detail

// 存储在 basic_memory_buffer 对象本身中的字符数，以避免动态内存分配
enum { inline_buffer_size = 500 };
/**
  \rst
  用于动态增长的内存缓冲区，适用于可以简单复制/构造的类型，其中前 ``SIZE`` 个元素存储在对象本身中。

  您可以使用以下常见字符类型的类型别名之一：

  +----------------+------------------------------+
  | 类型           | 定义                         |
  +================+==============================+
  | memory_buffer  | basic_memory_buffer<char>    |
  +----------------+------------------------------+
  | wmemory_buffer | basic_memory_buffer<wchar_t> |
  +----------------+------------------------------+

  **示例**::

     fmt::memory_buffer out;
     format_to(out, "The answer is {}.", 42);

  这将把以下输出附加到 ``out`` 对象：

  .. code-block:: none

     The answer is 42.

  输出可以使用 ``to_string(out)`` 转换为 ``std::string``。
  \endrst
 */
template <typename T, size_t SIZE = inline_buffer_size,
          typename Allocator = std::allocator<T>>
class basic_memory_buffer : public detail::buffer<T> {
 private:
  T store_[SIZE];

  // 不继承自 Allocator，避免为其生成 type_info。
  Allocator alloc_;

  // 释放缓冲区分配的内存。
  void deallocate() {
    T* data = this->data();
    if (data != store_) alloc_.deallocate(data, this->capacity());
  }

 protected:
  void grow(size_t size) final FMT_OVERRIDE;

 public:
  using value_type = T;
  using const_reference = const T&;

  explicit basic_memory_buffer(const Allocator& alloc = Allocator())
      : alloc_(alloc) {
    this->set(store_, SIZE);
  }
  ~basic_memory_buffer() { deallocate(); }

 private:
  // 从其他缓冲区移动数据到此缓冲区。
  void move(basic_memory_buffer& other) {
    alloc_ = std::move(other.alloc_);
    T* data = other.data();
    size_t size = other.size(), capacity = other.capacity();
    # 如果传入的数据指针和其他对象的存储指针相同
    if (data == other.store_) {
      # 将当前对象的存储指针设置为其他对象的存储指针，并复制数据
      this->set(store_, capacity);
      std::uninitialized_copy(other.store_, other.store_ + size,
                              detail::make_checked(store_, capacity));
    } else {
      # 否则，将当前对象的存储指针设置为传入的数据指针，并将其他对象的存储指针设置为内联数组的指针
      this->set(data, capacity);
      # 设置指向内联数组的指针，以便在释放内存时不调用删除操作
      other.set(other.store_, 0);
    }
    # 调整缓冲区大小
    this->resize(size);
  }

 public:
  /**
    \rst
    构造一个 :class:`fmt::basic_memory_buffer` 对象，将其他对象的内容移动到其中。
    \endrst
   */
  basic_memory_buffer(basic_memory_buffer&& other) FMT_NOEXCEPT { move(other); }

  /**
    \rst
    将其他 ``basic_memory_buffer`` 对象的内容移动到当前对象中。
    \endrst
   */
  basic_memory_buffer& operator=(basic_memory_buffer&& other) FMT_NOEXCEPT {
    # 断言当前对象和其他对象不相同
    FMT_ASSERT(this != &other, "");
    # 释放当前对象的内存
    deallocate();
    # 移动其他对象的内容到当前对象中
    move(other);
    return *this;
  }

  # 返回与该缓冲区关联的分配器的副本
  Allocator get_allocator() const { return alloc_; }

  /**
    调整缓冲区大小以包含 *count* 个元素。如果 T 是 POD 类型，则可能不会初始化新元素。
   */
  void resize(size_t count) { this->try_resize(count); }

  /** 增加缓冲区的容量到 *new_capacity*。 */
  void reserve(size_t new_capacity) { this->try_reserve(new_capacity); }

  # 直接将数据追加到缓冲区中
  using detail::buffer<T>::append;
  template <typename ContiguousRange>
  void append(const ContiguousRange& range) {
    append(range.data(), range.data() + range.size());
  }
};

template <typename T, size_t SIZE, typename Allocator>
void basic_memory_buffer<T, SIZE, Allocator>::grow(size_t size) {
#ifdef FMT_FUZZ
  // 如果处于模糊测试模式且需要增长的大小超过5000，则抛出运行时错误
  if (size > 5000) throw std::runtime_error("fuzz mode - won't grow that much");
#endif
  // 记录旧的容量
  size_t old_capacity = this->capacity();
  // 计算新的容量为旧容量的1.5倍
  size_t new_capacity = old_capacity + old_capacity / 2;
  // 如果需要的大小超过新容量，则将新容量设为需要的大小
  if (size > new_capacity) new_capacity = size;
  // 获取旧数据的指针
  T* old_data = this->data();
  // 分配新容量的内存
  T* new_data =
      std::allocator_traits<Allocator>::allocate(alloc_, new_capacity);
  // 复制旧数据到新内存，不会抛出异常
  std::uninitialized_copy(old_data, old_data + this->size(),
                          detail::make_checked(new_data, new_capacity));
  // 设置新数据和新容量
  this->set(new_data, new_capacity);
  // 根据标准，释放内存不应该抛出异常，即使抛出异常，缓冲区已经使用新存储并将在析构函数中释放
  if (old_data != store_) alloc_.deallocate(old_data, old_capacity);
}

using memory_buffer = basic_memory_buffer<char>;
using wmemory_buffer = basic_memory_buffer<wchar_t>;

template <typename T, size_t SIZE, typename Allocator>
struct is_contiguous<basic_memory_buffer<T, SIZE, Allocator>> : std::true_type {
};

/** A formatting error such as invalid format string. */
FMT_CLASS_API
class FMT_API format_error : public std::runtime_error {
 public:
  // 构造函数，接受 C 风格字符串作为参数
  explicit format_error(const char* message) : std::runtime_error(message) {}
  // 构造函数，接受 std::string 作为参数
  explicit format_error(const std::string& message)
      : std::runtime_error(message) {}
  format_error(const format_error&) = default;
  format_error& operator=(const format_error&) = default;
  format_error(format_error&&) = default;
  format_error& operator=(format_error&&) = default;
  // 析构函数
  ~format_error() FMT_NOEXCEPT FMT_OVERRIDE;
};

namespace detail {

template <typename T>
// 判断类型 T 是否为有符号类型
using is_signed =
    std::integral_constant<bool, std::numeric_limits<T>::is_signed ||
                                     std::is_same<T, int128_t>::value>;
// 如果值为负数则返回 true，否则返回 false。
// 与 `value < 0` 相同，但如果 T 是无符号类型则不会产生警告。
template <typename T, FMT_ENABLE_IF(is_signed<T>::value)>
FMT_CONSTEXPR bool is_negative(T value) {
  return value < 0;
}
// 如果值为负数则返回 true，否则返回 false。
// 与 `value < 0` 相同，但如果 T 是无符号类型则不会产生警告。
template <typename T, FMT_ENABLE_IF(!is_signed<T>::value)>
FMT_CONSTEXPR bool is_negative(T) {
  return false;
}

// 如果 T 是浮点类型并且被支持，则返回 true，否则返回 false。
template <typename T, FMT_ENABLE_IF(std::is_floating_point<T>::value)>
FMT_CONSTEXPR bool is_supported_floating_point(T) {
  return (std::is_same<T, float>::value && FMT_USE_FLOAT) ||
         (std::is_same<T, double>::value && FMT_USE_DOUBLE) ||
         (std::is_same<T, long double>::value && FMT_USE_LONG_DOUBLE);
}

// 表示所有整数类型 T 的值的最小的 uint32_t、uint64_t、uint128_t。
template <typename T>
using uint32_or_64_or_128_t =
    conditional_t<num_bits<T>() <= 32 && !FMT_REDUCE_INT_INSTANTIATIONS,
                  uint32_t,
                  conditional_t<num_bits<T>() <= 64, uint64_t, uint128_t>>;

// 内部使用的 128 位整数类型
struct FMT_EXTERN_TEMPLATE_API uint128_wrapper {
  uint128_wrapper() = default;

#if FMT_USE_INT128
  uint128_t internal_;

  uint128_wrapper(uint64_t high, uint64_t low) FMT_NOEXCEPT
      : internal_{static_cast<uint128_t>(low) |
                  (static_cast<uint128_t>(high) << 64)} {}

  uint128_wrapper(uint128_t u) : internal_{u} {}

  uint64_t high() const FMT_NOEXCEPT { return uint64_t(internal_ >> 64); }
  uint64_t low() const FMT_NOEXCEPT { return uint64_t(internal_); }

  uint128_wrapper& operator+=(uint64_t n) & FMT_NOEXCEPT {
    internal_ += n;
    return *this;
  }
#else
  // 定义一个64位无符号整数变量high_
  uint64_t high_;
  // 定义一个64位无符号整数变量low_
  uint64_t low_;

  // 构造函数，初始化high_和low_
  uint128_wrapper(uint64_t high, uint64_t low) FMT_NOEXCEPT : high_{high},
                                                              low_{low} {}

  // 返回high_的值
  uint64_t high() const FMT_NOEXCEPT { return high_; }
  // 返回low_的值
  uint64_t low() const FMT_NOEXCEPT { return low_; }

  // 重载+=运算符，对64位无符号整数进行加法操作
  uint128_wrapper& operator+=(uint64_t n) & FMT_NOEXCEPT {
#  if defined(_MSC_VER) && defined(_M_X64)
    // 如果是MSVC编译器并且是64位平台，则使用内置函数_addcarry_u64进行加法操作
    unsigned char carry = _addcarry_u64(0, low_, n, &low_);
    _addcarry_u64(carry, high_, 0, &high_);
    return *this;
#  else
    // 否则使用普通的加法操作
    uint64_t sum = low_ + n;
    high_ += (sum < low_ ? 1 : 0);
    low_ = sum;
    return *this;
#  endif
  }
#endif
};

// 用于内部使用的可除性测试的表项类型
template <typename T> struct FMT_EXTERN_TEMPLATE_API divtest_table_entry {
  T mod_inv;
  T max_quotient;
};

// 静态数据放置在这个类模板中，用于头文件配置
template <typename T = void> struct FMT_EXTERN_TEMPLATE_API basic_data {
  // 64位10的幂的数组
  static const uint64_t powers_of_10_64[];
  // 32位0或10的幂的数组
  static const uint32_t zero_or_powers_of_10_32[];
  // 64位0或10的幂的数组
  static const uint64_t zero_or_powers_of_10_64[];
  // grisu算法中10的幂的尾数数组
  static const uint64_t grisu_pow10_significands[];
  // grisu算法中10的幂的指数数组
  static const int16_t grisu_pow10_exponents[];
  // pow(5, n)的可除性测试表格，32位版本
  static const divtest_table_entry<uint32_t> divtest_table_for_pow5_32[];
  // pow(5, n)的可除性测试表格，64位版本
  static const divtest_table_entry<uint64_t> divtest_table_for_pow5_64[];
  // dragonbox算法中10的幂的尾数数组，64位版本
  static const uint64_t dragonbox_pow10_significands_64[];
  // dragonbox算法中10的幂的尾数数组，128位版本
  static const uint128_wrapper dragonbox_pow10_significands_128[];
  // log10(2)的尾数部分
  static const uint64_t log10_2_significand = 0x4d104d427de7fbcc;
  // 如果不使用完整缓存的dragonbox算法，则使用64位版本的5的幂的数组
  static const uint64_t powers_of_5_64[];
  // dragonbox算法中10的幂的恢复误差数组
  static const uint32_t dragonbox_pow10_recovery_errors[];
#endif
  // 使用 pairs 而不是 chars，因为 GCC 为 pairs 生成的代码略微更好。
  using digit_pair = char[2];
  // 定义常量数组 digits
  static const digit_pair digits[];
  // 定义常量数组 hex_digits
  static const char hex_digits[];
  // 定义常量数组 foreground_color
  static const char foreground_color[];
  // 定义常量数组 background_color
  static const char background_color[];
  // 定义常量数组 reset_color
  static const char reset_color[5];
  // 定义常量数组 wreset_color
  static const wchar_t wreset_color[5];
  // 定义常量数组 signs
  static const char signs[];
  // 定义常量数组 left_padding_shifts
  static const char left_padding_shifts[5];
  // 定义常量数组 right_padding_shifts
  static const char right_padding_shifts[5];
};

// 将 bsr(n) 映射到 ceil(log10(pow(2, bsr(n) + 1) - 1))。
// 这是一个函数而不是数组，以解决 GCC10 中的一个 bug (#1810)。
FMT_INLINE uint16_t bsr2log10(int bsr) {
  // 定义常量数组 data
  static constexpr uint16_t data[] = {
      1,  1,  1,  2,  2,  2,  3,  3,  3,  4,  4,  4,  4,  5,  5,  5,
      6,  6,  6,  7,  7,  7,  7,  8,  8,  8,  9,  9,  9,  10, 10, 10,
      10, 11, 11, 11, 12, 12, 12, 13, 13, 13, 13, 14, 14, 14, 15, 15,
      15, 16, 16, 16, 16, 17, 17, 17, 18, 18, 18, 19, 19, 19, 19, 20};
  return data[bsr];
}

#ifndef FMT_EXPORTED
// 实例化 basic_data<void> 模板
FMT_EXTERN template struct basic_data<void>;
#endif

// 这是一个结构体而不是别名，以避免在 gcc 中出现阴影警告。
struct data : basic_data<> {};

#ifdef FMT_BUILTIN_CLZLL
// 返回 n 中的十进制数字个数。不计入前导零，但当 n == 0 时，count_digits 返回 1。
inline int count_digits(uint64_t n) {
  // https://github.com/fmtlib/format-benchmark/blob/master/digits10
  // 计算 bsr2log10(FMT_BUILTIN_CLZLL(n | 1) ^ 63) 的值
  auto t = bsr2log10(FMT_BUILTIN_CLZLL(n | 1) ^ 63);
  // 返回结果
  return t - (n < data::zero_or_powers_of_10_64[t]);
}
#else
// 当 __builtin_clz 不可用时，使用的 count_digits 的回退版本。
inline int count_digits(uint64_t n) {
  int count = 1;
  for (;;) {
    // 整数除法很慢，因此对四个数字进行一组操作，而不是每个数字都进行操作。这个想法来自 Alexandrescu 的演讲 "Three Optimization Tips for C++"。参见 speed-test 进行比较。
    if (n < 10) return count;
    if (n < 100) return count + 1;
    if (n < 1000) return count + 2;
    # 如果 n 小于 10000，则返回 count + 3
    if (n < 10000) return count + 3;
    # 将 n 除以 10000，并将结果赋值给 n
    n /= 10000u;
    # count 加 4
    count += 4;
  }
// 结束条件宏定义
}
#endif

// 如果使用了 128 位整数，计算数字的位数
#if FMT_USE_INT128
inline int count_digits(uint128_t n) {
  int count = 1;
  for (;;) {
    // 整数除法很慢，所以对于一组四位数，而不是每个数字都进行整数除法。这个想法来自 Alexandrescu 的讲座"Three Optimization Tips for C++"。参见 speed-test 进行比较。
    if (n < 10) return count;
    if (n < 100) return count + 1;
    if (n < 1000) return count + 2;
    if (n < 10000) return count + 3;
    n /= 10000U;
    count += 4;
  }
}
#endif

// 计算 n 中的数字个数。BITS = log2(radix)。
template <unsigned BITS, typename UInt> inline int count_digits(UInt n) {
  int num_digits = 0;
  do {
    ++num_digits;
  } while ((n >>= BITS) != 0);
  return num_digits;
}

// 为 4 位整数特化 count_digits
template <> int count_digits<4>(detail::fallback_uintptr n);

// 根据不同的编译器定义 FMT_ALWAYS_INLINE 宏
#if FMT_GCC_VERSION || FMT_CLANG_VERSION
#  define FMT_ALWAYS_INLINE inline __attribute__((always_inline))
#elif FMT_MSC_VER
#  define FMT_ALWAYS_INLINE __forceinline
#else
#  define FMT_ALWAYS_INLINE inline
#endif

// 为了抑制不必要的安全 cookie 检查
#if FMT_MSC_VER && !FMT_CLANG_VERSION
#  define FMT_SAFEBUFFERS __declspec(safebuffers)
#else
#  define FMT_SAFEBUFFERS
#endif

// 如果定义了 FMT_BUILTIN_CLZ，则使用更好的性能在 32 位平台上的 count_digits 的可选版本
#ifdef FMT_BUILTIN_CLZ
inline int count_digits(uint32_t n) {
  auto t = bsr2log10(FMT_BUILTIN_CLZ(n | 1) ^ 31);
  return t - (n < data::zero_or_powers_of_10_32[t]);
}
#endif

// 返回整数类型的最大十进制位数
template <typename Int> constexpr int digits10() FMT_NOEXCEPT {
  return std::numeric_limits<Int>::digits10;
}
// 为 int128_t 类型特化 digits10
template <> constexpr int digits10<int128_t>() FMT_NOEXCEPT { return 38; }
// 为 uint128_t 类型特化 digits10
template <> constexpr int digits10<uint128_t>() FMT_NOEXCEPT { return 38; }

// 定义 grouping_impl 函数模板
template <typename Char> FMT_API std::string grouping_impl(locale_ref loc);
// 定义 grouping 函数模板
template <typename Char> inline std::string grouping(locale_ref loc) {
  return grouping_impl<char>(loc);
}
// 为 wchar_t 类型的分组规则模板特化，返回分组规则的实现
template <> inline std::string grouping<wchar_t>(locale_ref loc) {
  return grouping_impl<wchar_t>(loc);
}

// 为 Char 类型的千位分隔符规则模板特化，返回千位分隔符的实现
template <typename Char> FMT_API Char thousands_sep_impl(locale_ref loc);
template <typename Char> inline Char thousands_sep(locale_ref loc) {
  return Char(thousands_sep_impl<char>(loc));
}
// 为 wchar_t 类型的千位分隔符规则模板特化，返回千位分隔符的实现
template <> inline wchar_t thousands_sep(locale_ref loc) {
  return thousands_sep_impl<wchar_t>(loc);
}

// 为 Char 类型的小数点规则模板特化，返回小数点的实现
template <typename Char> FMT_API Char decimal_point_impl(locale_ref loc);
template <typename Char> inline Char decimal_point(locale_ref loc) {
  return Char(decimal_point_impl<char>(loc));
}
// 为 wchar_t 类型的小数点规则模板特化，返回小数点的实现
template <> inline wchar_t decimal_point(locale_ref loc) {
  return decimal_point_impl<wchar_t>(loc);
}

// 比较两个字符是否相等
template <typename Char> bool equal2(const Char* lhs, const char* rhs) {
  return lhs[0] == rhs[0] && lhs[1] == rhs[1];
}
// 比较两个字符是否相等
inline bool equal2(const char* lhs, const char* rhs) {
  return memcmp(lhs, rhs, 2) == 0;
}

// 从 src 复制两个字符到 dst
template <typename Char> void copy2(Char* dst, const char* src) {
  *dst++ = static_cast<Char>(*src++);
  *dst = static_cast<Char>(*src);
}
// 从 src 复制两个字符到 dst
inline void copy2(char* dst, const char* src) { memcpy(dst, src, 2); }

// 格式化一个十进制无符号整数值，写入指向指定大小缓冲区的 out 指针
// 调用者必须确保缓冲区足够大
template <typename Char, typename UInt>
inline format_decimal_result<Char*> format_decimal(Char* out, UInt value,
                                                   int size) {
  FMT_ASSERT(size >= count_digits(value), "invalid digit count");
  out += size;
  Char* end = out;
  while (value >= 100) {
    // 整数除法很慢，因此对两个数字进行分组，而不是每个数字都进行除法
    // 这个想法来自 Alexandrescu 的演讲 "Three Optimization Tips for C++"。参见 speed-test 进行比较。
    # 减去2，更新out的值
    out -= 2;
    # 将value的个位和十位数字拷贝到out指向的位置
    copy2(out, data::digits[value % 100]);
    # 更新value的值
    value /= 100;
  }
  # 如果value小于10，表示只有个位数
  if (value < 10) {
    # 将value转换为字符并存入out指向的位置，然后返回结果
    *--out = static_cast<Char>('0' + value);
    return {out, end};
  }
  # 如果value大于等于10，表示有两位数
  out -= 2;
  # 将value的十位和个位数字拷贝到out指向的位置
  copy2(out, data::digits[value]);
  # 返回结果
  return {out, end};
// 使用模板定义一个函数，根据输入的值和位数，将其格式化为十进制数并输出到迭代器中
template <typename Char, typename UInt, typename Iterator,
          FMT_ENABLE_IF(!std::is_pointer<remove_cvref_t<Iterator>>::value)>
inline format_decimal_result<Iterator> format_decimal(Iterator out, UInt value,
                                                      int num_digits) {
  // 缓冲区应该足够大，能够容纳所有的数字（<= digits10 + 1）
  enum { max_size = digits10<UInt>() + 1 };
  // 创建一个字符数组作为缓冲区
  Char buffer[2 * max_size];
  // 调用 format_decimal 函数，将格式化后的结果存储到缓冲区中，并返回结束位置
  auto end = format_decimal(buffer, value, num_digits).end;
  // 返回结果，包括输出迭代器和格式化后的数据
  return {out, detail::copy_str<Char>(buffer, end, out)};
}

// 使用模板定义一个函数，根据输入的值和位数，将其格式化为指定进制的数并输出到字符数组中
template <unsigned BASE_BITS, typename Char, typename UInt>
inline Char* format_uint(Char* buffer, UInt value, int num_digits,
                         bool upper = false) {
  // 将缓冲区指针移动到指定的位置
  buffer += num_digits;
  // 将结束指针指向当前位置
  Char* end = buffer;
  // 循环处理每一位数字
  do {
    // 根据进制选择对应的数字字符集
    const char* digits = upper ? "0123456789ABCDEF" : data::hex_digits;
    // 获取当前位的值
    unsigned digit = (value & ((1 << BASE_BITS) - 1));
    // 将当前位的值转换为字符并存储到缓冲区中
    *--buffer = static_cast<Char>(BASE_BITS < 4 ? static_cast<char>('0' + digit)
                                                : digits[digit]);
  } while ((value >>= BASE_BITS) != 0);
  // 返回结束位置指针
  return end;
}

// 使用模板定义一个函数，根据输入的值和位数，将其格式化为指定进制的数并输出到字符数组中
template <unsigned BASE_BITS, typename Char>
Char* format_uint(Char* buffer, detail::fallback_uintptr n, int num_digits,
                  bool = false) {
  // 计算每个字符能够存储的位数
  auto char_digits = std::numeric_limits<unsigned char>::digits / 4;
  // 计算起始位置
  int start = (num_digits + char_digits - 1) / char_digits - 1;
  // 如果起始位置有余数，则单独处理
  if (int start_digits = num_digits % char_digits) {
    // 获取起始位置的值，并将其格式化输出到缓冲区中
    unsigned value = n.value[start--];
    buffer = format_uint<BASE_BITS>(buffer, value, start_digits);
  }
  // 循环处理每一位数字
  for (; start >= 0; --start) {
    // 获取当前位置的值
    unsigned value = n.value[start];
    // 将缓冲区指针移动到指定的位置
    buffer += char_digits;
    // 逐个处理每个字符
    auto p = buffer;
    for (int i = 0; i < char_digits; ++i) {
      // 获取当前位的值
      unsigned digit = (value & ((1 << BASE_BITS) - 1));
      // 将当前位的值转换为字符并存储到缓冲区中
      *--p = static_cast<Char>(data::hex_digits[digit]);
      // 右移处理下一位
      value >>= BASE_BITS;
    }
  }
  // 返回结束位置指针
  return buffer;
}

// 使用模板定义一个函数，根据输入的值和位数，将其格式化为指定进制的数并输出到迭代器中
template <unsigned BASE_BITS, typename Char, typename It, typename UInt>
// 将无符号整数格式化为字符串，并将结果写入输出迭代器
inline It format_uint(It out, UInt value, int num_digits, bool upper = false) {
  // 缓冲区应该足够大，能够容纳所有数字（digits / BASE_BITS + 1）
  char buffer[num_bits<UInt>() / BASE_BITS + 1];
  // 调用 format_uint 函数将无符号整数格式化为字符串
  format_uint<BASE_BITS>(buffer, value, num_digits, upper);
  // 将格式化后的字符串复制到输出迭代器中
  return detail::copy_str<Char>(buffer, buffer + num_digits, out);
}

// 从 UTF-8 转换为 UTF-16 的转换器
class utf8_to_utf16 {
 private:
  wmemory_buffer buffer_;

 public:
  // 使用 UTF-8 字符串视图初始化转换器
  FMT_API explicit utf8_to_utf16(string_view s);
  // 转换为 UTF-16 字符串视图
  operator wstring_view() const { return {&buffer_[0], size()}; }
  // 返回缓冲区大小
  size_t size() const { return buffer_.size() - 1; }
  // 返回缓冲区指针
  const wchar_t* c_str() const { return &buffer_[0]; }
  // 返回 UTF-16 字符串
  std::wstring str() const { return {&buffer_[0], size()}; }
};

// 空类型模板
template <typename T = void> struct null {};

// 解决 gcc 4.8 中的数组初始化问题
template <typename Char> struct fill_t {
 private:
  enum { max_size = 4 };
  Char data_[max_size];
  unsigned char size_;

 public:
  // 将字符串视图赋值给 fill_t 对象
  FMT_CONSTEXPR void operator=(basic_string_view<Char> s) {
    auto size = s.size();
    if (size > max_size) {
      FMT_THROW(format_error("invalid fill"));
      return;
    }
    for (size_t i = 0; i < size; ++i) data_[i] = s[i];
    size_ = static_cast<unsigned char>(size);
  }

  // 返回大小
  size_t size() const { return size_; }
  // 返回数据指针
  const Char* data() const { return data_; }

  // 重载下标运算符
  FMT_CONSTEXPR Char& operator[](size_t index) { return data_[index]; }
  FMT_CONSTEXPR const Char& operator[](size_t index) const {
    return data_[index];
  }

  // 创建 fill_t 对象
  static FMT_CONSTEXPR fill_t<Char> make() {
    auto fill = fill_t<Char>();
    fill[0] = Char(' ');
    fill.size_ = 1;
    return fill;
  }
};
}  // namespace detail

// 由于 gcc 的 bug，无法将枚举类用作位域
namespace align {
// 对齐类型枚举
enum type { none, left, right, center, numeric };
}
// 对齐类型别名
using align_t = align::type;

namespace sign {
// 符号类型枚举
enum type { none, minus, plus, space };
}
// 符号类型别名
using sign_t = sign::type;
// 用于内置类型和字符串类型的格式说明符
template <typename Char> struct basic_format_specs {
  int width;  // 宽度
  int precision;  // 精度
  char type;  // 类型
  align_t align : 4;  // 对齐方式
  sign_t sign : 3;  // 符号
  bool alt : 1;  // 是否使用备用形式 ('#')
  detail::fill_t<Char> fill;  // 填充字符

  // 构造函数，初始化默认值
  constexpr basic_format_specs()
      : width(0),
        precision(-1),
        type(0),
        align(align::none),
        sign(sign::none),
        alt(false),
        fill(detail::fill_t<Char>::make()) {}
};

// 使用 char 类型的基本格式说明符
using format_specs = basic_format_specs<char>;

namespace detail {
namespace dragonbox {

// Dragonbox 使用的特定于类型的信息
template <class T> struct float_info;

// float 类型的信息
template <> struct float_info<float> {
  using carrier_uint = uint32_t;  // 无符号整数类型
  static const int significand_bits = 23;  // 尾数位数
  static const int exponent_bits = 8;  // 指数位数
  static const int min_exponent = -126;  // 最小指数
  static const int max_exponent = 127;  // 最大指数
  static const int exponent_bias = -127;  // 指数偏移
  static const int decimal_digits = 9;  // 十进制位数
  static const int kappa = 1;  // kappa 值
  static const int big_divisor = 100;  // 大除数
  static const int small_divisor = 10;  // 小除数
  static const int min_k = -31;  // 最小 k 值
  static const int max_k = 46;  // 最大 k 值
  static const int cache_bits = 64;  // 缓存位数
  static const int divisibility_check_by_5_threshold = 39;  // 除以 5 的可整除性检查阈值
  static const int case_fc_pm_half_lower_threshold = -1;  // case_fc_pm_half 的下限阈值
  static const int case_fc_pm_half_upper_threshold = 6;  // case_fc_pm_half 的上限阈值
  static const int case_fc_lower_threshold = -2;  // case_fc 的下限阈值
  static const int case_fc_upper_threshold = 6;  // case_fc 的上限阈值
  static const int case_shorter_interval_left_endpoint_lower_threshold = 2;  // case_shorter_interval_left_endpoint 的下限阈值
  static const int case_shorter_interval_left_endpoint_upper_threshold = 3;  // case_shorter_interval_left_endpoint 的上限阈值
  static const int shorter_interval_tie_lower_threshold = -35;  // shorter_interval_tie 的下限阈值
  static const int shorter_interval_tie_upper_threshold = -35;  // shorter_interval_tie 的上限阈值
  static const int max_trailing_zeros = 7;  // 最大尾随零数
};
// 定义一个模板结构体，用于存储双精度浮点数的信息
template <> struct float_info<double> {
  using carrier_uint = uint64_t;  // 使用 uint64_t 作为载体类型
  static const int significand_bits = 52;  // 尾数位数
  static const int exponent_bits = 11;  // 指数位数
  static const int min_exponent = -1022;  // 最小指数
  static const int max_exponent = 1023;  // 最大指数
  static const int exponent_bias = -1023;  // 指数偏移
  static const int decimal_digits = 17;  // 十进制位数
  static const int kappa = 2;  // kappa 值
  static const int big_divisor = 1000;  // 大除数
  static const int small_divisor = 100;  // 小除数
  static const int min_k = -292;  // 最小 k 值
  static const int max_k = 326;  // 最大 k 值
  static const int cache_bits = 128;  // 缓存位数
  static const int divisibility_check_by_5_threshold = 86;  // 可被 5 整除的阈值
  static const int case_fc_pm_half_lower_threshold = -2;  // case_fc_pm_half 的下限阈值
  static const int case_fc_pm_half_upper_threshold = 9;  // case_fc_pm_half 的上限阈值
  static const int case_fc_lower_threshold = -4;  // case_fc 的下限阈值
  static const int case_fc_upper_threshold = 9;  // case_fc 的上限阈值
  static const int case_shorter_interval_left_endpoint_lower_threshold = 2;  // case_shorter_interval_left_endpoint 的下限阈值
  static const int case_shorter_interval_left_endpoint_upper_threshold = 3;  // case_shorter_interval_left_endpoint 的上限阈值
  static const int shorter_interval_tie_lower_threshold = -77;  // shorter_interval_tie 的下限阈值
  static const int shorter_interval_tie_upper_threshold = -77;  // shorter_interval_tie 的上限阈值
  static const int max_trailing_zeros = 16;  // 最大尾随零数
};

// 定义一个模板结构体，用于存储十进制浮点数的信息
template <typename T> struct decimal_fp {
  using significand_type = typename float_info<T>::carrier_uint;  // 使用 float_info 中的 carrier_uint 作为尾数类型
  significand_type significand;  // 尾数
  int exponent;  // 指数
};

// 将浮点数转换为十进制浮点数
template <typename T> decimal_fp<T> to_decimal(T x) FMT_NOEXCEPT;
}  // namespace dragonbox

// 浮点数的表示格式枚举
enum class float_format : unsigned char {
  general,  // 通用格式：基于幅度的指数表示法或固定点表示法
  exp,      // 指数表示法，精度默认为 6，例如 1.2e-3
  fixed,    // 固定点表示法，精度默认为 6，例如 0.0012
  hex
};

// 浮点数的规格
struct float_specs {
  int precision;  // 精度
  float_format format : 8;  // 格式
  sign_t sign : 8;  // 符号
  bool upper : 1;  // 是否大写
  bool locale : 1;  // 是否使用本地化
  bool binary32 : 1;  // 是否是二进制32位
  bool use_grisu : 1;  // 是否使用 grisu 算法
  bool showpoint : 1;  // 是否显示小数点
};

// 将指数 exp 以 "[+-]d{2,3}" 的形式写入缓冲区
// 以指定的字符类型和迭代器类型写入指数值
template <typename Char, typename It> It write_exponent(int exp, It it) {
  // 断言指数范围在 -10000 到 10000 之间
  FMT_ASSERT(-10000 < exp && exp < 10000, "exponent out of range");
  // 如果指数小于 0，则写入负号并取绝对值
  if (exp < 0) {
    *it++ = static_cast<Char>('-');
    exp = -exp;
  } else {
    *it++ = static_cast<Char>('+');
  }
  // 如果指数大于等于 100，则写入对应的字符
  if (exp >= 100) {
    const char* top = data::digits[exp / 100];
    if (exp >= 1000) *it++ = static_cast<Char>(top[0]);
    *it++ = static_cast<Char>(top[1]);
    exp %= 100;
  }
  // 写入指数的个位和十位数字
  const char* d = data::digits[exp];
  *it++ = static_cast<Char>(d[0]);
  *it++ = static_cast<Char>(d[1]);
  return it;
}

// 格式化浮点数值为字符串
template <typename T>
int format_float(T value, int precision, float_specs specs, buffer<char>& buf);

// 使用 snprintf 格式化浮点数值为字符串
template <typename T>
int snprintf_float(T value, int precision, float_specs specs,
                   buffer<char>& buf);

// 提升浮点数类型
template <typename T> T promote_float(T value) { return value; }
inline double promote_float(float value) { return static_cast<double>(value); }

// 处理整数类型的格式化规范
template <typename Handler>
FMT_CONSTEXPR void handle_int_type_spec(char spec, Handler&& handler) {
  switch (spec) {
  case 0:
  case 'd':
    handler.on_dec();
    break;
  case 'x':
  case 'X':
    handler.on_hex();
    break;
  case 'b':
  case 'B':
    handler.on_bin();
    break;
  case 'o':
    handler.on_oct();
    break;
#ifdef FMT_DEPRECATED_N_SPECIFIER
  case 'n':
#endif
  case 'L':
    handler.on_num();
    break;
  case 'c':
    handler.on_chr();
    break;
  default:
    handler.on_error();
  }
}

// 解析浮点数类型的格式化规范
template <typename ErrorHandler = error_handler, typename Char>
FMT_CONSTEXPR float_specs parse_float_type_spec(
    const basic_format_specs<Char>& specs, ErrorHandler&& eh = {}) {
  auto result = float_specs();
  result.showpoint = specs.alt;
  switch (specs.type) {
  case 0:
    result.format = float_format::general;
    result.showpoint |= specs.precision > 0;
    break;
  case 'G':
    result.upper = true;
    FMT_FALLTHROUGH;
  case 'g':
    result.format = float_format::general;
  // 其他情况
    # 如果遇到 'E'，则将结果的大小写设置为大写
    break;
  # 如果遇到 'E'，则将结果的大小写设置为大写，并且继续执行下面的代码
  case 'E':
    result.upper = true;
    FMT_FALLTHROUGH;
  # 如果遇到 'e'，则将结果的格式设置为指数格式
  case 'e':
    result.format = float_format::exp;
    # 如果精度不为0，则显示小数点
    result.showpoint |= specs.precision != 0;
    break;
  # 如果遇到 'F'，则将结果的大小写设置为大写
  case 'F':
    result.upper = true;
    FMT_FALLTHROUGH;
  # 如果遇到 'f'，则将结果的格式设置为固定格式
  case 'f':
    result.format = float_format::fixed;
    # 如果精度不为0，则显示小数点
    result.showpoint |= specs.precision != 0;
    break;
  # 如果遇到 'A'，则将结果的大小写设置为大写
  case 'A':
    result.upper = true;
    FMT_FALLTHROUGH;
  # 如果遇到 'a'，则将结果的格式设置为十六进制格式
  case 'a':
    result.format = float_format::hex;
    break;
#ifdef FMT_DEPRECATED_N_SPECIFIER
  // 如果定义了 FMT_DEPRECATED_N_SPECIFIER，则执行以下代码
  case 'n':
#endif
  // 当前情况为 'n' 或 'L' 时
  case 'L':
    // 设置结果的 locale 属性为 true
    result.locale = true;
    // 结束当前情况的处理
    break;
  // 其他情况
  default:
    // 调用错误处理器的 on_error 方法，传入错误信息 "invalid type specifier"
    eh.on_error("invalid type specifier");
    // 结束当前情况的处理
    break;
  }
  // 返回结果
  return result;
}

template <typename Char, typename Handler>
FMT_CONSTEXPR void handle_char_specs(const basic_format_specs<Char>* specs,
                                     Handler&& handler) {
  // 如果未提供格式规范，则调用处理器的 on_char 方法并返回
  if (!specs) return handler.on_char();
  // 如果提供了类型并且类型不是 'c'，则调用处理器的 on_int 方法并返回
  if (specs->type && specs->type != 'c') return handler.on_int();
  // 如果对齐方式为数字、有符号位或者使用了替代形式，则调用处理器的 on_error 方法，传入错误信息 "invalid format specifier for char"
  if (specs->align == align::numeric || specs->sign != sign::none || specs->alt)
    handler.on_error("invalid format specifier for char");
  // 调用处理器的 on_char 方法
  handler.on_char();
}

template <typename Char, typename Handler>
FMT_CONSTEXPR void handle_cstring_type_spec(Char spec, Handler&& handler) {
  // 如果类型规范为 0 或 's'，则调用处理器的 on_string 方法
  if (spec == 0 || spec == 's')
    handler.on_string();
  // 如果类型规范为 'p'，则调用处理器的 on_pointer 方法
  else if (spec == 'p')
    handler.on_pointer();
  // 否则，调用处理器的 on_error 方法，传入错误信息 "invalid type specifier"
  else
    handler.on_error("invalid type specifier");
}

template <typename Char, typename ErrorHandler>
FMT_CONSTEXPR void check_string_type_spec(Char spec, ErrorHandler&& eh) {
  // 如果类型规范不为 0 且不为 's'，则调用错误处理器的 on_error 方法，传入错误信息 "invalid type specifier"
  if (spec != 0 && spec != 's') eh.on_error("invalid type specifier");
}

template <typename Char, typename ErrorHandler>
FMT_CONSTEXPR void check_pointer_type_spec(Char spec, ErrorHandler&& eh) {
  // 如果类型规范不为 0 且不为 'p'，则调用错误处理器的 on_error 方法，传入错误信息 "invalid type specifier"
  if (spec != 0 && spec != 'p') eh.on_error("invalid type specifier");
}

template <typename ErrorHandler> class int_type_checker : private ErrorHandler {
 public:
  // 构造函数，接受一个错误处理器作为参数
  FMT_CONSTEXPR explicit int_type_checker(ErrorHandler eh) : ErrorHandler(eh) {}

  // 下面的方法都是空实现，不做任何操作
  FMT_CONSTEXPR void on_dec() {}
  FMT_CONSTEXPR void on_hex() {}
  FMT_CONSTEXPR void on_bin() {}
  FMT_CONSTEXPR void on_oct() {}
  FMT_CONSTEXPR void on_num() {}
  FMT_CONSTEXPR void on_chr() {}

  // 当出现错误时，调用错误处理器的 on_error 方法，传入错误信息 "invalid type specifier"
  FMT_CONSTEXPR void on_error() {
    ErrorHandler::on_error("invalid type specifier");
  }
};

template <typename ErrorHandler>
// 定义一个类 char_specs_checker，继承自 ErrorHandler
class char_specs_checker : public ErrorHandler {
 private:
  // 声明一个私有成员变量 type_
  char type_;

 public:
  // 构造函数，接受一个字符类型和一个 ErrorHandler 对象
  FMT_CONSTEXPR char_specs_checker(char type, ErrorHandler eh)
      : ErrorHandler(eh), type_(type) {}

  // 处理整数类型的方法
  FMT_CONSTEXPR void on_int() {
    // 调用 handle_int_type_spec 方法处理整数类型的规范
    handle_int_type_spec(type_, int_type_checker<ErrorHandler>(*this));
  }
  // 处理字符类型的方法
  FMT_CONSTEXPR void on_char() {}
};

// 定义一个模板类 cstring_type_checker，继承自 ErrorHandler
template <typename ErrorHandler>
class cstring_type_checker : public ErrorHandler {
 public:
  // 构造函数，接受一个 ErrorHandler 对象
  FMT_CONSTEXPR explicit cstring_type_checker(ErrorHandler eh)
      : ErrorHandler(eh) {}

  // 处理字符串类型的方法
  FMT_CONSTEXPR void on_string() {}
  // 处理指针类型的方法
  FMT_CONSTEXPR void on_pointer() {}
};

// 填充输出内容，根据填充规范和数量
template <typename OutputIt, typename Char>
FMT_NOINLINE OutputIt fill(OutputIt it, size_t n, const fill_t<Char>& fill) {
  // 获取填充字符的大小
  auto fill_size = fill.size();
  // 如果填充字符大小为1，则直接填充 n 个填充字符
  if (fill_size == 1) return std::fill_n(it, n, fill[0]);
  // 否则，循环填充 n 个填充字符
  for (size_t i = 0; i < n; ++i) it = std::copy_n(fill.data(), fill_size, it);
  return it;
}

// 根据格式规范在输出内容上进行填充
template <align::type align = align::left, typename OutputIt, typename Char,
          typename F>
inline OutputIt write_padded(OutputIt out,
                             const basic_format_specs<Char>& specs, size_t size,
                             size_t width, const F& f) {
  // 静态断言，检查对齐方式是否为左对齐或右对齐
  static_assert(align == align::left || align == align::right, "");
  // 将规范中的宽度转换为无符号整数
  unsigned spec_width = to_unsigned(specs.width);
  // 计算填充数量
  size_t padding = spec_width > width ? spec_width - width : 0;
  // 根据对齐方式选择左填充还是右填充
  auto* shifts = align == align::left ? data::left_padding_shifts
                                      : data::right_padding_shifts;
  size_t left_padding = padding >> shifts[specs.align];
  // 预留输出空间
  auto it = reserve(out, size + padding * specs.fill.size());
  // 左填充
  it = fill(it, left_padding, specs.fill);
  // 执行函数 f，写入输出内容
  it = f(it);
  // 右填充
  it = fill(it, padding - left_padding, specs.fill);
  return base_iterator(out, it);
}
// 写入带填充的内容，根据对齐方式和格式规范，使用指定的函数对象 f
template <align::type align = align::left, typename OutputIt, typename Char,
          typename F>
inline OutputIt write_padded(OutputIt out,
                             const basic_format_specs<Char>& specs, size_t size,
                             const F& f) {
  // 调用另一个重载的 write_padded 函数，传入相同的参数和额外的 size 参数
  return write_padded<align>(out, specs, size, size, f);
}

// 写入字节内容，根据格式规范，使用指定的输出迭代器 out
template <typename Char, typename OutputIt>
OutputIt write_bytes(OutputIt out, string_view bytes,
                     const basic_format_specs<Char>& specs) {
  // 定义迭代器类型 iterator，使用 remove_reference_t 移除引用
  using iterator = remove_reference_t<decltype(reserve(out, 0))>;
  // 调用 write_padded 函数，传入指定的参数和 lambda 表达式
  return write_padded(out, specs, bytes.size(), [bytes](iterator it) {
    // 获取字节数据的指针
    const char* data = bytes.data();
    // 将字节数据拷贝到输出迭代器 it 中
    return copy_str<Char>(data, data + bytes.size(), it);
  });
}

// 写入整数的数据结构，不依赖于输出迭代器类型，用于避免模板代码膨胀
template <typename Char> struct write_int_data {
  size_t size; // 整数数据的大小
  size_t padding; // 填充大小

  // 构造函数，根据数字位数、前缀和格式规范初始化数据结构
  write_int_data(int num_digits, string_view prefix,
                 const basic_format_specs<Char>& specs)
      : size(prefix.size() + to_unsigned(num_digits)), padding(0) {
    if (specs.align == align::numeric) {
      auto width = to_unsigned(specs.width);
      if (width > size) {
        padding = width - size;
        size = width;
      }
    } else if (specs.precision > num_digits) {
      size = prefix.size() + to_unsigned(specs.precision);
      padding = to_unsigned(specs.precision - num_digits);
    }
  }
}

// 写入整数，格式为 <left-padding><prefix><numeric-padding><digits><right-padding>
// 其中 <digits> 由函数对象 f(it) 写入
template <typename OutputIt, typename Char, typename F>
OutputIt write_int(OutputIt out, int num_digits, string_view prefix,
                   const basic_format_specs<Char>& specs, F f) {
  // 初始化 write_int_data 结构体
  auto data = write_int_data<Char>(num_digits, prefix, specs);
  // 定义迭代器类型 iterator，使用 remove_reference_t 移除引用
  using iterator = remove_reference_t<decltype(reserve(out, 0))>;
  // 调用 write_padded 函数，传入指定的参数和 lambda 表达式
  return write_padded<align::right>(out, specs, data.size, [=](iterator it) {
    # 如果前缀的大小不为0，则将前缀内容复制到迭代器指向的位置
    if (prefix.size() != 0)
      it = copy_str<Char>(prefix.begin(), prefix.end(), it);
    # 使用指定字符填充迭代器指向的位置，填充的次数为data.padding
    it = std::fill_n(it, data.padding, static_cast<Char>('0'));
    # 返回调用f函数的结果
    return f(it);
  });
  // 写入函数，将字符串视图 s 写入到输出迭代器 out 中，根据 specs 指定的格式进行处理
  template <typename StrChar, typename Char, typename OutputIt>
  OutputIt write(OutputIt out, basic_string_view<StrChar> s,
                 const basic_format_specs<Char>& specs) {
    // 获取字符串数据的指针和大小
    auto data = s.data();
    auto size = s.size();
    // 如果指定了精度并且小于字符串大小，则将 size 调整为指定精度的字符数
    if (specs.precision >= 0 && to_unsigned(specs.precision) < size)
      size = code_point_index(s, to_unsigned(specs.precision));
    // 如果指定了宽度，则根据字符数调整宽度
    auto width = specs.width != 0
                     ? count_code_points(basic_string_view<StrChar>(data, size))
                     : 0;
    // 定义迭代器类型
    using iterator = remove_reference_t<decltype(reserve(out, 0))>;
    // 调用 write_padded 函数，根据 specs 和 size、width 写入数据
    return write_padded(out, specs, size, width, [=](iterator it) {
      return copy_str<Char>(data, data + size, it);
    });
  }

  // 处理整数类型规范的处理程序，用于写入整数
  template <typename OutputIt, typename Char, typename UInt> struct int_writer {
    OutputIt out;
    locale_ref locale;
    const basic_format_specs<Char>& specs;
    UInt abs_value;
    char prefix[4];
    unsigned prefix_size;

    using iterator =
        remove_reference_t<decltype(reserve(std::declval<OutputIt&>(), 0))>;

    // 获取前缀
    string_view get_prefix() const { return string_view(prefix, prefix_size); }

    // 构造函数，初始化成员变量
    template <typename Int>
    int_writer(OutputIt output, locale_ref loc, Int value,
               const basic_format_specs<Char>& s)
        : out(output),
          locale(loc),
          specs(s),
          abs_value(static_cast<UInt>(value)),
          prefix_size(0) {
      static_assert(std::is_same<uint32_or_64_or_128_t<Int>, UInt>::value, "");
      // 如果是负数，则添加负号前缀
      if (is_negative(value)) {
        prefix[0] = '-';
        ++prefix_size;
        abs_value = 0 - abs_value;
      } else if (specs.sign != sign::none && specs.sign != sign::minus) {
        prefix[0] = specs.sign == sign::plus ? '+' : ' ';
        ++prefix_size;
      }
    }

    // 处理十进制整数
    void on_dec() {
      // 计算整数的位数
      auto num_digits = count_digits(abs_value);
    }
  }
  // 调用 write_int 函数，将整数转换为字符串并写入输出流
  out = write_int(
      out, num_digits, get_prefix(), specs, [this, num_digits](iterator it) {
        // 使用 format_decimal 函数将绝对值转换为指定位数的十进制字符串
        return format_decimal<Char>(it, abs_value, num_digits).end;
      });
}

// 处理十六进制格式
void on_hex() {
  // 如果设置了替代格式标志，添加十六进制前缀
  if (specs.alt) {
    prefix[prefix_size++] = '0';
    prefix[prefix_size++] = specs.type;
  }
  // 计算十六进制数的位数
  int num_digits = count_digits<4>(abs_value);
  // 调用 write_int 函数，将整数转换为十六进制字符串并写入输出流
  out = write_int(out, num_digits, get_prefix(), specs,
                  [this, num_digits](iterator it) {
                    // 使用 format_uint 函数将绝对值转换为指定位数的十六进制字符串
                    return format_uint<4, Char>(it, abs_value, num_digits,
                                                specs.type != 'x');
                  });
}

// 处理二进制格式
void on_bin() {
  // 如果设置了替代格式标志，添加二进制前缀
  if (specs.alt) {
    prefix[prefix_size++] = '0';
    prefix[prefix_size++] = static_cast<char>(specs.type);
  }
  // 计算二进制数的位数
  int num_digits = count_digits<1>(abs_value);
  // 调用 write_int 函数，将整数转换为二进制字符串并写入输出流
  out = write_int(out, num_digits, get_prefix(), specs,
                  [this, num_digits](iterator it) {
                    // 使用 format_uint 函数将绝对值转换为指定位数的二进制字符串
                    return format_uint<1, Char>(it, abs_value, num_digits);
                  });
}

// 处理八进制格式
void on_oct() {
  // 计算八进制数的位数
  int num_digits = count_digits<3>(abs_value);
  // 如果设置了替代格式标志且精度小于等于位数且绝对值不为0，添加八进制前缀
  if (specs.alt && specs.precision <= num_digits && abs_value != 0) {
    prefix[prefix_size++] = '0';
  }
  // 调用 write_int 函数，将整数转换为八进制字符串并写入输出流
  out = write_int(out, num_digits, get_prefix(), specs,
                  [this, num_digits](iterator it) {
                    // 使用 format_uint 函数将绝对值转换为指定位数的八进制字符串
                    return format_uint<3, Char>(it, abs_value, num_digits);
                  });
}

// 定义分隔符大小为1
enum { sep_size = 1 };

// 处理数字格式
void on_num() {
  // 获取本地化环境中的分组规则
  std::string groups = grouping<Char>(locale);
  // 如果分组规则为空，使用默认的十进制格式
  if (groups.empty()) return on_dec();
  // 获取本地化环境中的千位分隔符
  auto sep = thousands_sep<Char>(locale);
  // 如果千位分隔符为空，使用默认的十进制格式
  if (!sep) return on_dec();
  // 计算整数的位数
  int num_digits = count_digits(abs_value);
  int size = num_digits, n = num_digits;
  // 获取分组规则的迭代器
  std::string::const_iterator group = groups.cbegin();
    // 当 group 不是 groups 的结尾迭代器，并且 n 大于 *group，并且 *group 大于 0，并且 *group 不等于 char 的最大值时，执行循环
    while (group != groups.cend() && n > *group && *group > 0 &&
           *group != max_value<char>()) {
      // size 增加 sep_size
      size += sep_size;
      // n 减去 *group
      n -= *group;
      // group 向后移动一个位置
      ++group;
    }
    // 如果 group 等于 groups 的结尾迭代器，size 增加 sep_size 乘以 ((n - 1) 除以 groups 的最后一个元素)
    if (group == groups.cend()) size += sep_size * ((n - 1) / groups.back());
    // 创建一个包含 40 个字符的数组 digits
    char digits[40];
    // 将 abs_value 转换为包含 num_digits 位的十进制数，并存储到 digits 数组中
    format_decimal(digits, abs_value, num_digits);
    // 创建一个基本内存缓冲区 buffer
    basic_memory_buffer<Char> buffer;
    // size 增加 prefix_size 的整数部分
    size += static_cast<int>(prefix_size);
    // 将 size 转换为无符号整数 usize，并将 buffer 的大小调整为 usize
    const auto usize = to_unsigned(size);
    buffer.resize(usize);
    // 创建一个基本字符串视图 s，包含 sep 和 sep_size
    basic_string_view<Char> s(&sep, sep_size);
    // 最不重要的十进制数字的索引，最不重要的数字的索引为 0
    int digit_index = 0;
    // 将 group 重置为 groups 的起始迭代器
    group = groups.cbegin();
    // 创建一个指针 p，指向 buffer 中的第 size 个元素
    auto p = buffer.data() + size;
    // 从最高位数字开始向最低位数字遍历
    for (int i = num_digits - 1; i >= 0; --i) {
      // 将 digits[i] 赋值给 *p，然后将 p 向前移动一个位置
      *--p = static_cast<Char>(digits[i]);
      // 如果 *group 小于等于 0，或者 digit_index 除以 *group 的余数不等于 0，或者 *group 等于 char 的最大值，则继续循环
      if (*group <= 0 || ++digit_index % *group != 0 ||
          *group == max_value<char>())
        continue;
      // 如果 group 不是 groups 的倒数第二个元素，则将 digit_index 重置为 0，并将 group 向后移动一个位置
      if (group + 1 != groups.cend()) {
        digit_index = 0;
        ++group;
      }
      // 将 p 向前移动 s 的大小个位置，并将 s 复制到 p 所指向的位置
      p -= s.size();
      std::uninitialized_copy(s.data(), s.data() + s.size(),
                              make_checked(p, s.size()));
    }
    // 如果 prefix_size 不等于 0，则将 p 所指向的前一个位置赋值为 '-'
    if (prefix_size != 0) p[-1] = static_cast<Char>('-');
    // 创建一个指向 buffer 数据的指针 data
    auto data = buffer.data();
    // 将 buffer 中的数据写入到 out 中，并根据 specs 和 usize 进行对齐
    out = write_padded<align::right>(
        out, specs, usize, usize,
        [=](iterator it) { return copy_str<Char>(data, data + size, it); });
  }

  // 将 abs_value 转换为字符并写入到 out 中
  void on_chr() { *out++ = static_cast<Char>(abs_value); }

  // 抛出格式错误异常
  FMT_NORETURN void on_error() {
    FMT_THROW(format_error("invalid type specifier"));
  }
// 写入非有限浮点数的字符串表示，根据是否为无穷大和规格参数来确定字符串内容
template <typename Char, typename OutputIt>
OutputIt write_nonfinite(OutputIt out, bool isinf,
                         const basic_format_specs<Char>& specs,
                         const float_specs& fspecs) {
  // 根据是否为无穷大和规格参数来确定字符串内容
  auto str =
      isinf ? (fspecs.upper ? "INF" : "inf") : (fspecs.upper ? "NAN" : "nan");
  // 字符串大小为3
  constexpr size_t str_size = 3;
  // 获取符号
  auto sign = fspecs.sign;
  // 计算字符串大小
  auto size = str_size + (sign ? 1 : 0);
  // 定义迭代器类型
  using iterator = remove_reference_t<decltype(reserve(out, 0))>;
  // 调用 write_padded 函数，写入格式化后的字符串
  return write_padded(out, specs, size, [=](iterator it) {
    // 如果有符号，写入符号
    if (sign) *it++ = static_cast<Char>(data::signs[sign]);
    // 返回写入字符串后的迭代器
    return copy_str<Char>(str, str + str_size, it);
  });
}

// 一个十进制浮点数的尾数 * pow(10, exp)
struct big_decimal_fp {
  const char* significand;
  int significand_size;
  int exp;
};

// 写入浮点数的字符串表示
template <typename OutputIt, typename Char>
OutputIt write_float(OutputIt out, const big_decimal_fp& fp, float_specs specs,
                     Char decimal_point) {
  // 获取尾数
  const char* digits = fp.significand;
  // 定义字符 '0'
  const Char zero = static_cast<Char>('0');

  // 计算输出指数
  int output_exp = fp.exp + fp.significand_size - 1;
  // 判断是否使用指数格式
  auto use_exp_format = [=]() {
    if (specs.format == float_format::exp) return true;
    if (specs.format != float_format::general) return false;
    // 格式化数字，如果指数在 [exp_lower, exp_upper) 范围内，则使用固定表示法
    const int exp_lower = -4, exp_upper = 16;
    return output_exp < exp_lower ||
           output_exp >= (specs.precision > 0 ? specs.precision : exp_upper);
  };
  // 如果使用指数格式
  if (use_exp_format()) {
    // 在第一个数字后插入小数点，并添加指数
    *out++ = static_cast<Char>(*digits);
    // 计算需要补零的个数
    int num_zeros = specs.precision - fp.significand_size;
    // 如果尾数大于1或者显示小数点，则写入小数点
    if (fp.significand_size > 1 || specs.showpoint) *out++ = decimal_point;
    // 写入尾数
    out = copy_str<Char>(digits + 1, digits + fp.significand_size, out);
    // 如果需要补零且显示小数点，则填充零
    if (num_zeros > 0 && specs.showpoint)
      out = std::fill_n(out, num_zeros, zero);
    // 根据规范中的 upper 属性确定指数部分的大写字母 'E' 或小写字母 'e'
    *out++ = static_cast<Char>(specs.upper ? 'E' : 'e');
    // 调用 write_exponent 函数将指数部分写入输出流
    return write_exponent<Char>(output_exp, out);
  }

  // 计算指数部分的值
  int exp = fp.exp + fp.significand_size;
  // 如果尾数部分小于等于指数部分
  if (fp.significand_size <= exp) {
    // 将尾数部分的数字复制到输出流中
    out = copy_str<Char>(digits, digits + fp.significand_size, out);
    // 在输出流中填充指定数量的零，以补齐指数部分
    out = std::fill_n(out, exp - fp.significand_size, zero);
    // 如果规范中要求显示小数点
    if (specs.showpoint) {
      // 在输出流中添加小数点
      *out++ = decimal_point;
      // 计算需要补齐的零的数量
      int num_zeros = specs.precision - exp;
      // 如果需要补齐的零的数量小于等于 0
      if (num_zeros <= 0) {
        // 如果规范中的格式不是固定格式，则在输出流中添加一个零
        if (specs.format != float_format::fixed) *out++ = zero;
        // 返回输出流
        return out;
      }
#ifdef FMT_FUZZ
      // 如果在模糊模式下，避免过多的 CPU 使用
      if (num_zeros > 5000)
        throw std::runtime_error("fuzz mode - avoiding excessive cpu use");
#endif
      // 将输出迭代器 out 填充 num_zeros 个 zero
      out = std::fill_n(out, num_zeros, zero);
    }
  } else if (exp > 0) {
    // 1234e-2 -> 12.34[0+]
    // 将指数部分的数字复制到输出迭代器 out
    out = copy_str<Char>(digits, digits + exp, out);
    if (!specs.showpoint) {
      // 如果不需要显示小数点
      if (fp.significand_size != exp) *out++ = decimal_point;
      return copy_str<Char>(digits + exp, digits + fp.significand_size, out);
    }
    *out++ = decimal_point;
    // 将小数部分的数字复制到输出迭代器 out
    out = copy_str<Char>(digits + exp, digits + fp.significand_size, out);
    // 添加尾部的零
    if (specs.precision > fp.significand_size)
      out = std::fill_n(out, specs.precision - fp.significand_size, zero);
  } else {
    // 1234e-6 -> 0.001234
    *out++ = zero;
    int num_zeros = -exp;
    // 如果尾部的零的数量不为零，或者尾部的零的数量不为零，或者需要显示小数点
    if (fp.significand_size == 0 && specs.precision >= 0 &&
        specs.precision < num_zeros) {
      num_zeros = specs.precision;
    }
    if (num_zeros != 0 || fp.significand_size != 0 || specs.showpoint) {
      *out++ = decimal_point;
      // 将输出迭代器 out 填充 num_zeros 个 zero
      out = std::fill_n(out, num_zeros, zero);
      // 将小数部分的数字复制到输出迭代器 out
      out = copy_str<Char>(digits, digits + fp.significand_size, out);
    }
  }
  return out;
}

// 给定数字 v = significand * pow(10, exp)。
template <typename OutputIt, typename Char>
OutputIt write_float(OutputIt out, const buffer<char>& significand, int exp,
                     const basic_format_specs<Char>& specs, float_specs fspecs,
                     Char decimal_point) {
  auto fp = big_decimal_fp{significand.data(),
                           static_cast<int>(significand.size()), exp};
  auto size =
      write_float(counting_iterator(), fp, fspecs, decimal_point).count();
  size += fspecs.sign ? 1 : 0;
  using iterator = remove_reference_t<decltype(reserve(out, 0))>;
  // 写入填充后的数字到输出迭代器 out
  return write_padded<align::right>(out, specs, size, [&](iterator it) {
    if (fspecs.sign) *it++ = static_cast<Char>(data::signs[fspecs.sign]);
    return write_float(it, fp, fspecs, decimal_point);
  });
}
template <typename Char, typename OutputIt, typename T,
          FMT_ENABLE_IF(std::is_floating_point<T>::value)>
OutputIt write(OutputIt out, T value, basic_format_specs<Char> specs,
               locale_ref loc = {}) {
  // 如果值不是支持的浮点数类型，则直接返回输出迭代器
  if (const_check(!is_supported_floating_point(value))) return out;
  // 解析浮点数类型的格式规范
  float_specs fspecs = parse_float_type_spec(specs);
  // 将规范中的符号赋给浮点数规范
  fspecs.sign = specs.sign;
  // 如果值为负数，则将符号设置为负号，并取绝对值
  if (std::signbit(value)) {  // value < 0 is false for NaN so use signbit.
    fspecs.sign = sign::minus;
    value = -value;
  } else if (fspecs.sign == sign::minus) {
    fspecs.sign = sign::none;
  }

  // 如果值不是有限的浮点数，则返回非有限数的写入结果
  if (!std::isfinite(value))
    return write_nonfinite(out, std::isinf(value), specs, fspecs);

  // 如果规范中对齐方式为数字，并且有符号，则在输出迭代器中预留一个位置用于符号，并更新规范
  if (specs.align == align::numeric && fspecs.sign) {
    auto it = reserve(out, 1);
    *it++ = static_cast<Char>(data::signs[fspecs.sign]);
    out = base_iterator(out, it);
    fspecs.sign = sign::none;
    if (specs.width != 0) --specs.width;
  }

  // 创建内存缓冲区
  memory_buffer buffer;
  // 如果浮点数格式为十六进制，则将值格式化为十六进制并写入输出迭代器
  if (fspecs.format == float_format::hex) {
    if (fspecs.sign) buffer.push_back(data::signs[fspecs.sign]);
    snprintf_float(promote_float(value), specs.precision, fspecs, buffer);
    return write_bytes(out, {buffer.data(), buffer.size()}, specs);
  }
  // 如果精度大于等于0或者类型不为0，则将精度设置为规范中的精度，否则设置为6
  int precision = specs.precision >= 0 || !specs.type ? specs.precision : 6;
  // 如果浮点数格式为指数形式，则根据精度调整指数
  if (fspecs.format == float_format::exp) {
    if (precision == max_value<int>())
      FMT_THROW(format_error("number is too big"));
    else
      ++precision;
  }
  // 如果值的类型为float，则将二进制32位标志设置为true
  if (const_check(std::is_same<T, float>())) fspecs.binary32 = true;
  // 根据值的类型判断是否使用grisu算法
  fspecs.use_grisu = is_fast_float<T>();
  // 格式化浮点数并返回指数
  int exp = format_float(promote_float(value), precision, fspecs, buffer);
  fspecs.precision = precision;
  // 根据规范中的小数点设置小数点字符
  Char point =
      fspecs.locale ? decimal_point<Char>(loc) : static_cast<Char>('.');
  // 将格式化后的浮点数写入输出迭代器
  return write_float(out, buffer, exp, specs, fspecs, point);
}

template <
    typename Char, typename OutputIt, typename T,
    FMT_ENABLE_IF(std::is_floating_point<T>::value&& is_fast_float<T>::value)>
// 写入浮点数值到输出流，根据值的特性进行格式化处理
OutputIt write(OutputIt out, T value) {
  // 如果值不是支持的浮点数类型，则直接返回输出流
  if (const_check(!is_supported_floating_point(value))) return out;
  // 获取浮点数的格式规范
  auto fspecs = float_specs();
  // 如果值为负数，则修改符号位并取绝对值
  if (std::signbit(value)) {  // value < 0 is false for NaN so use signbit.
    fspecs.sign = sign::minus;
    value = -value;
  }

  // 创建基本的格式规范
  auto specs = basic_format_specs<Char>();
  // 如果值不是有限的浮点数，则调用相应的非有限数值处理函数
  if (!std::isfinite(value))
    return write_nonfinite(out, std::isinf(value), specs, fspecs);

  // 根据值的类型选择合适的处理类型
  using type = conditional_t<std::is_same<T, long double>::value, double, T>;
  // 将浮点数值转换为十进制表示
  auto dec = dragonbox::to_decimal(static_cast<type>(value));
  // 创建内存缓冲区
  memory_buffer buf;
  // 将十进制表示的浮点数值写入缓冲区
  write<char>(buffer_appender<char>(buf), dec.significand);
  // 将格式化后的浮点数值写入输出流
  return write_float(out, buf, dec.exponent, specs, fspecs,
                     static_cast<Char>('.'));
}

// 当值为浮点数且不是快速浮点数时，调用此函数进行写入
template <typename Char, typename OutputIt, typename T,
          FMT_ENABLE_IF(std::is_floating_point<T>::value &&
                        !is_fast_float<T>::value)>
inline OutputIt write(OutputIt out, T value) {
  return write(out, value, basic_format_specs<Char>());
}

// 写入字符到输出流，并根据格式规范进行填充
template <typename Char, typename OutputIt>
OutputIt write_char(OutputIt out, Char value,
                    const basic_format_specs<Char>& specs) {
  using iterator = remove_reference_t<decltype(reserve(out, 0))>;
  return write_padded(out, specs, 1, [=](iterator it) {
    *it++ = value;
    return it;
  });
}

// 写入指针值到输出流，并根据格式规范进行填充
template <typename Char, typename OutputIt, typename UIntPtr>
OutputIt write_ptr(OutputIt out, UIntPtr value,
                   const basic_format_specs<Char>* specs) {
  // 计算指针值的位数
  int num_digits = count_digits<4>(value);
  auto size = to_unsigned(num_digits) + size_t(2);
  using iterator = remove_reference_t<decltype(reserve(out, 0))>;
  // 定义写入函数
  auto write = [=](iterator it) {
    *it++ = static_cast<Char>('0');
    *it++ = static_cast<Char>('x');
    return format_uint<4, Char>(it, value, num_digits);
  };
  // 根据是否存在格式规范选择相应的填充方式
  return specs ? write_padded<align::right>(out, *specs, size, write)
               : base_iterator(out, write(reserve(out, size)));
}
// 定义模板结构体，用于判断类型是否为整数型
template <typename T> struct is_integral : std::is_integral<T> {};
// 特化模板结构体，将 int128_t 类型定义为 true_type
template <> struct is_integral<int128_t> : std::true_type {};
// 特化模板结构体，将 uint128_t 类型定义为 true_type
template <> struct is_integral<uint128_t> : std::true_type {};

// 定义模板函数，用于处理 monostate 类型的数据
template <typename Char, typename OutputIt>
OutputIt write(OutputIt out, monostate) {
  // 断言，如果条件为假，则终止程序执行
  FMT_ASSERT(false, "");
  return out;
}

// 定义模板函数，用于处理 string_view 类型的数据
template <typename Char, typename OutputIt,
          FMT_ENABLE_IF(!std::is_same<Char, char>::value)>
OutputIt write(OutputIt out, string_view value) {
  // 为输出迭代器预留足够的空间
  auto it = reserve(out, value.size());
  // 将字符串拷贝到输出迭代器指向的位置
  it = copy_str<Char>(value.begin(), value.end(), it);
  return base_iterator(out, it);
}

// 定义模板函数，用于处理 basic_string_view 类型的数据
template <typename Char, typename OutputIt>
OutputIt write(OutputIt out, basic_string_view<Char> value) {
  // 为输出迭代器预留足够的空间
  auto it = reserve(out, value.size());
  // 将字符串拷贝到输出迭代器指向的位置
  it = std::copy(value.begin(), value.end(), it);
  return base_iterator(out, it);
}

// 定义模板函数，用于处理 buffer_appender<Char> 类型的数据
template <typename Char>
buffer_appender<Char> write(buffer_appender<Char> out,
                            basic_string_view<Char> value) {
  // 将字符串追加到输出容器中
  get_container(out).append(value.begin(), value.end());
  return out;
}

// 定义模板函数，用于处理整数类型数据
template <typename Char, typename OutputIt, typename T,
          FMT_ENABLE_IF(is_integral<T>::value &&
                        !std::is_same<T, bool>::value &&
                        !std::is_same<T, Char>::value)>
OutputIt write(OutputIt out, T value) {
  // 将值转换为无符号整数类型
  auto abs_value = static_cast<uint32_or_64_or_128_t<T>>(value);
  // 判断是否为负数
  bool negative = is_negative(value);
  // 计算数字位数
  int num_digits = count_digits(abs_value);
  // 计算输出的大小
  auto size = (negative ? 1 : 0) + static_cast<size_t>(num_digits);
  // 为输出迭代器预留足够的空间
  auto it = reserve(out, size);
  // 如果可以直接转换为指针，则直接进行格式化输出
  if (auto ptr = to_pointer<Char>(it, size)) {
    if (negative) *ptr++ = static_cast<Char>('-');
    format_decimal<Char>(ptr, abs_value, num_digits);
    return out;
  }
  // 如果无法直接转换为指针，则按字符逐个输出
  if (negative) *it++ = static_cast<Char>('-');
  it = format_decimal<Char>(it, abs_value, num_digits).end;
  return base_iterator(out, it);
}
// 用于将布尔值格式化为字符串并写入输出迭代器
template <typename Char, typename OutputIt>
OutputIt write(OutputIt out, bool value) {
  return write<Char>(out, string_view(value ? "true" : "false"));
}

// 用于将单个字符写入输出迭代器
template <typename Char, typename OutputIt>
OutputIt write(OutputIt out, Char value) {
  auto it = reserve(out, 1);  // 为输出分配空间
  *it++ = value;  // 将字符写入输出迭代器
  return base_iterator(out, it);  // 返回更新后的输出迭代器
}

// 用于将字符串写入输出迭代器
template <typename Char, typename OutputIt>
OutputIt write(OutputIt out, const Char* value) {
  if (!value) {
    FMT_THROW(format_error("string pointer is null"));  // 如果字符串指针为空，则抛出异常
  } else {
    auto length = std::char_traits<Char>::length(value);  // 获取字符串长度
    out = write(out, basic_string_view<Char>(value, length));  // 将字符串写入输出迭代器
  }
  return out;  // 返回更新后的输出迭代器
}

// 用于将指针写入输出迭代器
template <typename Char, typename OutputIt>
OutputIt write(OutputIt out, const void* value) {
  return write_ptr<Char>(out, to_uintptr(value), nullptr);  // 将指针写入输出迭代器
}

// 用于格式化自定义类型并将其写入输出迭代器
template <typename Char, typename OutputIt, typename T>
auto write(OutputIt out, const T& value) -> typename std::enable_if<
    mapped_type_constant<T, basic_format_context<OutputIt, Char>>::value ==
        type::custom_type,
    OutputIt>::type {
  using context_type = basic_format_context<OutputIt, Char>;
  using formatter_type =
      conditional_t<has_formatter<T, context_type>::value,
                    typename context_type::template formatter_type<T>,
                    fallback_formatter<T, Char>>;
  context_type ctx(out, {}, {});
  return formatter_type().format(value, ctx);  // 格式化自定义类型并将其写入输出迭代器
}

// 用于格式化参数并通过输出迭代器写入
// 这是一个类，而不是通用lambda，以便与C++11兼容
template <typename OutputIt, typename Char> struct default_arg_formatter {
  using context = basic_format_context<OutputIt, Char>;

  OutputIt out;
  basic_format_args<context> args;
  locale_ref loc;

  template <typename T> OutputIt operator()(T value) {
    return write<Char>(out, value);  // 格式化参数并将其写入输出迭代器
  }

  OutputIt operator()(typename basic_format_arg<context>::handle handle) {
    basic_format_parse_context<Char> parse_ctx({});  // 创建解析上下文
    # 创建基本格式上下文对象，用于格式化输出
    basic_format_context<OutputIt, Char> format_ctx(out, args, loc);
    # 使用格式化上下文对象处理格式化操作
    handle.format(parse_ctx, format_ctx);
    # 返回格式化后的输出结果
    return format_ctx.out();
  }
  // 结束了一个类的定义
};

// 定义了一个模板类 arg_formatter_base，接受一个输出迭代器类型 OutputIt，一个字符类型 Char，一个错误处理器类型 ErrorHandler
template <typename OutputIt, typename Char,
          typename ErrorHandler = error_handler>
class arg_formatter_base {
 public:
  // 定义了一些类型别名
  using iterator = OutputIt;
  using char_type = Char;
  using format_specs = basic_format_specs<Char>;

 private:
  // 定义了一些私有成员变量
  iterator out_;
  locale_ref locale_;
  format_specs* specs_;

  // 尝试在输出范围中预留 n 个额外字符的空间
  // 返回一个指向预留范围的指针或者 out_ 的引用
  auto reserve(size_t n) -> decltype(detail::reserve(out_, n)) {
    return detail::reserve(out_, n);
  }

  // 定义了一个类型别名 reserve_iterator
  using reserve_iterator = remove_reference_t<decltype(
      detail::reserve(std::declval<iterator&>(), 0))>;

  // 定义了一个写入整数的函数模板
  template <typename T> void write_int(T value, const format_specs& spec) {
    using uint_type = uint32_or_64_or_128_t<T>;
    int_writer<iterator, Char, uint_type> w(out_, locale_, value, spec);
    handle_int_type_spec(spec.type, w);
    out_ = w.out;
  }

  // 定义了一个写入字符的函数
  void write(char value) {
    auto&& it = reserve(1);
    *it++ = value;
  }

  // 定义了一个写入字符的函数模板
  template <typename Ch, FMT_ENABLE_IF(std::is_same<Ch, Char>::value)>
  void write(Ch value) {
    out_ = detail::write<Char>(out_, value);
  }

  // 定义了一个写入字符串视图的函数
  void write(string_view value) {
    auto&& it = reserve(value.size());
    it = copy_str<Char>(value.begin(), value.end(), it);
  }

  // 定义了一个写入宽字符串视图的函数
  void write(wstring_view value) {
    static_assert(std::is_same<Char, wchar_t>::value, "");
    auto&& it = reserve(value.size());
    it = std::copy(value.begin(), value.end(), it);
  }

  // 定义了一个写入字符数组的函数模板
  template <typename Ch>
  void write(const Ch* s, size_t size, const format_specs& specs) {
    auto width = specs.width != 0
                     ? count_code_points(basic_string_view<Ch>(s, size))
                     : 0;
    out_ = write_padded(out_, specs, size, width, [=](reserve_iterator it) {
      return copy_str<Char>(s, s + size, it);
    });
  }

  // 定义了一个写入字符串视图的函数模板
  template <typename Ch>
  void write(basic_string_view<Ch> s, const format_specs& specs = {}) {
    // 调用 detail 命名空间中的 write 函数，将 s 写入 out_ 中，并使用 specs 进行格式化
    out_ = detail::write(out_, s, specs);
  }

  // 写入指针的函数
  void write_pointer(const void* p) {
    // 调用 write_ptr 函数，将 p 转换为 uintptr 后写入 out_ 中，并使用 specs_ 进行格式化
    out_ = write_ptr<char_type>(out_, to_uintptr(p), specs_);
  }

  // 处理字符类型的结构体
  struct char_spec_handler : ErrorHandler {
    arg_formatter_base& formatter;
    Char value;

    char_spec_handler(arg_formatter_base& f, Char val)
        : formatter(f), value(val) {}

    void on_int() {
      // 如果有 specs，则将 value 转换为 int 后写入 out_ 中
      formatter.write_int(static_cast<int>(value), *formatter.specs_);
    }
    void on_char() {
      // 如果有 specs，则将 value 写入 out_ 中，并使用 specs 进行格式化，否则直接写入 value
      if (formatter.specs_)
        formatter.out_ = write_char(formatter.out_, value, *formatter.specs_);
      else
        formatter.write(value);
    }
  };

  // 处理字符串类型的结构体
  struct cstring_spec_handler : error_handler {
    arg_formatter_base& formatter;
    const Char* value;

    cstring_spec_handler(arg_formatter_base& f, const Char* val)
        : formatter(f), value(val) {}

    void on_string() { 
      // 写入字符串 value
      formatter.write(value); 
    }
    void on_pointer() { 
      // 写入指针 value
      formatter.write_pointer(value); 
    }
  };

 protected:
  // 返回 out_ 的迭代器
  iterator out() { return out_; }
  // 返回 specs_ 的指针
  format_specs* specs() { return specs_; }

  // 写入布尔值的函数
  void write(bool value) {
    // 如果有 specs，则根据 value 写入 "true" 或 "false"，并使用 specs 进行格式化，否则直接写入 value
    if (specs_)
      write(string_view(value ? "true" : "false"), *specs_);
    else
      out_ = detail::write<Char>(out_, value);
  }

  // 写入字符指针的函数
  void write(const Char* value) {
    if (!value) {
      // 如果 value 为空指针，则抛出格式化错误
      FMT_THROW(format_error("string pointer is null"));
    } else {
      // 否则，根据 value 的长度创建字符串视图 sv，如果有 specs，则使用 specs 进行格式化，否则直接写入 sv
      auto length = std::char_traits<char_type>::length(value);
      basic_string_view<char_type> sv(value, length);
      specs_ ? write(sv, *specs_) : write(sv);
    }
  }

 public:
  // 构造函数，初始化 out_、locale_ 和 specs_
  arg_formatter_base(OutputIt out, format_specs* s, locale_ref loc)
      : out_(out), locale_(loc), specs_(s) {}

  // 处理空类型的函数
  iterator operator()(monostate) {
    // 抛出无效参数类型的错误
    FMT_ASSERT(false, "invalid argument type");
    return out_;
  }

  // 处理整数类型的函数
  template <typename T, FMT_ENABLE_IF(is_integral<T>::value)>
  FMT_INLINE iterator operator()(T value) {
    // 如果有 specs，则根据 value 写入 out_ 中，并使用 specs 进行格式化，否则直接写入 value
    if (specs_)
      write_int(value, *specs_);
    else
      out_ = detail::write<Char>(out_, value);
  // 返回 out_ 变量
  return out_;
}

// 处理字符类型的参数
iterator operator()(Char value) {
  // 处理字符类型的参数规格
  handle_char_specs(specs_,
                    char_spec_handler(*this, static_cast<Char>(value)));
  // 返回 out_ 变量
  return out_;
}

// 处理布尔类型的参数
iterator operator()(bool value) {
  // 如果有参数规格且规格类型存在，则调用相应的处理函数
  if (specs_ && specs_->type) return (*this)(value ? 1 : 0);
  // 写入布尔值
  write(value != 0);
  // 返回 out_ 变量
  return out_;
}

// 处理浮点类型的参数
template <typename T, FMT_ENABLE_IF(std::is_floating_point<T>::value)>
iterator operator()(T value) {
  // 获取参数规格
  auto specs = specs_ ? *specs_ : format_specs();
  // 如果浮点数类型受支持，则写入值
  if (const_check(is_supported_floating_point(value)))
    out_ = detail::write(out_, value, specs, locale_);
  else
    // 抛出错误，表示不支持的浮点数类型
    FMT_ASSERT(false, "unsupported float argument type");
  // 返回 out_ 变量
  return out_;
}

// 处理以字符指针形式传入的参数
iterator operator()(const Char* value) {
  // 如果没有参数规格，则直接写入值
  if (!specs_) return write(value), out_;
  // 处理字符指针类型的参数规格
  handle_cstring_type_spec(specs_->type, cstring_spec_handler(*this, value));
  // 返回 out_ 变量
  return out_;
}

// 处理以字符视图形式传入的参数
iterator operator()(basic_string_view<Char> value) {
  // 如果有参数规格，则检查字符串类型规格并写入值
  if (specs_) {
    check_string_type_spec(specs_->type, error_handler());
    write(value, *specs_);
  } else {
    // 否则直接写入值
    write(value);
  }
  // 返回 out_ 变量
  return out_;
}

// 处理以 void 指针形式传入的参数
iterator operator()(const void* value) {
  // 如果有参数规格，则检查指针类型规格
  if (specs_) check_pointer_type_spec(specs_->type, error_handler());
  // 写入指针值
  write_pointer(value);
  // 返回 out_ 变量
  return out_;
}
// 检查字符是否为合法的变量名起始字符，包括大小写字母和下划线
template <typename Char> FMT_CONSTEXPR bool is_name_start(Char c) {
  return ('a' <= c && c <= 'z') || ('A' <= c && c <= 'Z') || '_' == c;
}

// 解析范围 [begin, end) 中的字符作为无符号整数。该函数假定范围非空，并且第一个字符是数字。
template <typename Char, typename ErrorHandler>
FMT_CONSTEXPR int parse_nonnegative_int(const Char*& begin, const Char* end,
                                        ErrorHandler&& eh) {
  FMT_ASSERT(begin != end && '0' <= *begin && *begin <= '9', "");
  unsigned value = 0;
  // 转换为无符号整数以防止警告。
  constexpr unsigned max_int = max_value<int>();
  unsigned big = max_int / 10;
  do {
    // 检查溢出。
    if (value > big) {
      value = max_int + 1;
      break;
    }
    value = value * 10 + unsigned(*begin - '0');
    ++begin;
  } while (begin != end && '0' <= *begin && *begin <= '9');
  if (value > max_int) eh.on_error("number is too big");
  return static_cast<int>(value);
}

// 自定义格式化器类模板
template <typename Context> class custom_formatter {
 private:
  using char_type = typename Context::char_type;

  basic_format_parse_context<char_type>& parse_ctx_;
  Context& ctx_;

 public:
  explicit custom_formatter(basic_format_parse_context<char_type>& parse_ctx,
                            Context& ctx)
      : parse_ctx_(parse_ctx), ctx_(ctx) {}

  // 重载函数调用运算符，用于格式化参数
  bool operator()(typename basic_format_arg<Context>::handle h) const {
    h.format(parse_ctx_, ctx_);
    return true;
  }

  // 重载函数调用运算符，用于处理其他类型的参数
  template <typename T> bool operator()(T) const { return false; }
};

// 检查类型是否为整数类型
template <typename T>
using is_integer =
    bool_constant<is_integral<T>::value && !std::is_same<T, bool>::value &&
                  !std::is_same<T, char>::value &&
                  !std::is_same<T, wchar_t>::value>;
// 定义一个模板类 width_checker，用于检查宽度参数是否合法
template <typename ErrorHandler> class width_checker {
 public:
  // 构造函数，接受一个 ErrorHandler 对象的引用
  explicit FMT_CONSTEXPR width_checker(ErrorHandler& eh) : handler_(eh) {}

  // 模板函数，接受一个整数类型的参数，并返回一个无符号长整型
  template <typename T, FMT_ENABLE_IF(is_integer<T>::value)>
  FMT_CONSTEXPR unsigned long long operator()(T value) {
    // 如果参数是负数，调用错误处理器的 on_error 方法
    if (is_negative(value)) handler_.on_error("negative width");
    // 返回参数的无符号长整型值
    return static_cast<unsigned long long>(value);
  }

  // 模板函数，接受一个非整数类型的参数，并返回一个无符号长整型
  template <typename T, FMT_ENABLE_IF(!is_integer<T>::value)>
  FMT_CONSTEXPR unsigned long long operator()(T) {
    // 调用错误处理器的 on_error 方法
    handler_.on_error("width is not integer");
    // 返回 0
    return 0;
  }

 private:
  // 成员变量，保存错误处理器的引用
  ErrorHandler& handler_;
};

// 定义一个模板类 precision_checker，用于检查精度参数是否合法
template <typename ErrorHandler> class precision_checker {
 public:
  // 构造函数，接受一个 ErrorHandler 对象的引用
  explicit FMT_CONSTEXPR precision_checker(ErrorHandler& eh) : handler_(eh) {}

  // 模板函数，接受一个整数类型的参数，并返回一个无符号长整型
  template <typename T, FMT_ENABLE_IF(is_integer<T>::value)>
  FMT_CONSTEXPR unsigned long long operator()(T value) {
    // 如果参数是负数，调用错误处理器的 on_error 方法
    if (is_negative(value)) handler_.on_error("negative precision");
    // 返回参数的无符号长整型值
    return static_cast<unsigned long long>(value);
  }

  // 模板函数，接受一个非整数类型的参数，并返回一个无符号长整型
  template <typename T, FMT_ENABLE_IF(!is_integer<T>::value)>
  FMT_CONSTEXPR unsigned long long operator()(T) {
    // 调用错误处理器的 on_error 方法
    handler_.on_error("precision is not integer");
    // 返回 0
    return 0;
  }

 private:
  // 成员变量，保存错误处理器的引用
  ErrorHandler& handler_;
};

// 定义一个格式规范设置类 specs_setter，用于设置 basic_format_specs 中的字段
template <typename Char> class specs_setter {
 public:
  // 构造函数，接受一个 basic_format_specs 对象的引用
  explicit FMT_CONSTEXPR specs_setter(basic_format_specs<Char>& specs)
      : specs_(specs) {}

  // 复制构造函数
  FMT_CONSTEXPR specs_setter(const specs_setter& other)
      : specs_(other.specs_) {}

  // 设置对齐方式
  FMT_CONSTEXPR void on_align(align_t align) { specs_.align = align; }
  // 设置填充字符
  FMT_CONSTEXPR void on_fill(basic_string_view<Char> fill) {
    specs_.fill = fill;
  }
  // 设置正号标记
  FMT_CONSTEXPR void on_plus() { specs_.sign = sign::plus; }
  // 设置负号标记
  FMT_CONSTEXPR void on_minus() { specs_.sign = sign::minus; }
  // 设置空格标记
  FMT_CONSTEXPR void on_space() { specs_.sign = sign::space; }
  // 设置替换标记
  FMT_CONSTEXPR void on_hash() { specs_.alt = true; }

  // 设置零填充标记
  FMT_CONSTEXPR void on_zero() {
    specs_.align = align::numeric;
    # 将索引为0的位置填充为字符'0'
    specs_.fill[0] = Char('0');
  }

  # 设置格式规范中的宽度
  FMT_CONSTEXPR void on_width(int width) { specs_.width = width; }
  # 设置格式规范中的精度
  FMT_CONSTEXPR void on_precision(int precision) {
    specs_.precision = precision;
  }
  # 结束设置格式规范中的精度
  FMT_CONSTEXPR void end_precision() {}

  # 设置格式规范中的类型
  FMT_CONSTEXPR void on_type(Char type) {
    specs_.type = static_cast<char>(type);
  }

  # 保护成员变量，用于存储格式规范
  protected:
  basic_format_specs<Char>& specs_;
  // 数字格式检查器模板类，用于检查格式说明符是否与参数类型一致
template <typename ErrorHandler> class numeric_specs_checker {
 public:
  // 构造函数，初始化错误处理器和参数类型
  FMT_CONSTEXPR numeric_specs_checker(ErrorHandler& eh, detail::type arg_type)
      : error_handler_(eh), arg_type_(arg_type) {}

  // 检查是否需要数字参数
  FMT_CONSTEXPR void require_numeric_argument() {
    if (!is_arithmetic_type(arg_type_))
      error_handler_.on_error("format specifier requires numeric argument");
  }

  // 检查符号
  FMT_CONSTEXPR void check_sign() {
    require_numeric_argument();
    if (is_integral_type(arg_type_) && arg_type_ != type::int_type &&
        arg_type_ != type::long_long_type && arg_type_ != type::char_type) {
      error_handler_.on_error("format specifier requires signed argument");
    }
  }

  // 检查精度
  FMT_CONSTEXPR void check_precision() {
    if (is_integral_type(arg_type_) || arg_type_ == type::pointer_type)
      error_handler_.on_error("precision not allowed for this argument type");
  }

 private:
  ErrorHandler& error_handler_;
  detail::type arg_type_;
};

// 一个格式说明符处理器，用于检查格式说明符是否与参数类型一致
template <typename Handler> class specs_checker : public Handler {
 private:
  numeric_specs_checker<Handler> checker_;

  // 抑制 MSVC 关于在初始化列表中使用 this 的警告
  FMT_CONSTEXPR Handler& error_handler() { return *this; }

 public:
  // 构造函数，初始化处理器和参数类型
  FMT_CONSTEXPR specs_checker(const Handler& handler, detail::type arg_type)
      : Handler(handler), checker_(error_handler(), arg_type) {}

  // 复制构造函数
  FMT_CONSTEXPR specs_checker(const specs_checker& other)
      : Handler(other), checker_(error_handler(), other.arg_type_) {}

  // 对齐处理
  FMT_CONSTEXPR void on_align(align_t align) {
    if (align == align::numeric) checker_.require_numeric_argument();
    Handler::on_align(align);
  }

  // 正号处理
  FMT_CONSTEXPR void on_plus() {
    checker_.check_sign();
    Handler::on_plus();
  }

  // 负号处理
  FMT_CONSTEXPR void on_minus() {
    checker_.check_sign();
    Handler::on_minus();
  }

  // 空格处理
  FMT_CONSTEXPR void on_space() {
    checker_.check_sign();
  # 调用Handler类的on_space方法
  Handler::on_space();
  # 调用Handler类的on_hash方法，要求参数为数字类型
  FMT_CONSTEXPR void on_hash() {
    checker_.require_numeric_argument();
    Handler::on_hash();
  }
  # 调用Handler类的on_zero方法，要求参数为数字类型
  FMT_CONSTEXPR void on_zero() {
    checker_.require_numeric_argument();
    Handler::on_zero();
  }
  # 结束精度设置，检查精度是否符合要求
  FMT_CONSTEXPR void end_precision() { checker_.check_precision(); }
};

// 模板函数，用于获取动态规格
template <template <typename> class Handler, typename FormatArg,
          typename ErrorHandler>
FMT_CONSTEXPR int get_dynamic_spec(FormatArg arg, ErrorHandler eh) {
  // 通过访问格式参数获取值
  unsigned long long value = visit_format_arg(Handler<ErrorHandler>(eh), arg);
  // 如果值大于 int 类型的最大值，则触发错误处理器
  if (value > to_unsigned(max_value<int>())) eh.on_error("number is too big");
  // 将值转换为 int 类型并返回
  return static_cast<int>(value);
}

// 自动 ID 结构体
struct auto_id {};

// 获取参数函数模板
template <typename Context, typename ID>
FMT_CONSTEXPR typename Context::format_arg get_arg(Context& ctx, ID id) {
  // 获取上下文中的参数
  auto arg = ctx.arg(id);
  // 如果参数不存在，则触发错误处理器
  if (!arg) ctx.on_error("argument not found");
  // 返回参数
  return arg;
}

// 带检查的标准格式规格处理器
template <typename ParseContext, typename Context>
class specs_handler : public specs_setter<typename Context::char_type> {
 public:
  using char_type = typename Context::char_type;

  // 构造函数
  FMT_CONSTEXPR specs_handler(basic_format_specs<char_type>& specs,
                              ParseContext& parse_ctx, Context& ctx)
      : specs_setter<char_type>(specs),
        parse_context_(parse_ctx),
        context_(ctx) {}

  // 处理动态宽度
  template <typename Id> FMT_CONSTEXPR void on_dynamic_width(Id arg_id) {
    // 设置规格的宽度为动态获取的值
    this->specs_.width = get_dynamic_spec<width_checker>(
        get_arg(arg_id), context_.error_handler());
  }

  // 处理动态精度
  template <typename Id> FMT_CONSTEXPR void on_dynamic_precision(Id arg_id) {
    // 设置规格的精度为动态获取的值
    this->specs_.precision = get_dynamic_spec<precision_checker>(
        get_arg(arg_id), context_.error_handler());
  }

  // 错误处理函数
  void on_error(const char* message) { context_.on_error(message); }

 private:
  // 仅用于与 gcc 4.4 兼容
  using format_arg = typename Context::format_arg;

  // 获取参数函数重载，根据自动 ID 获取参数
  FMT_CONSTEXPR format_arg get_arg(auto_id) {
    return detail::get_arg(context_, parse_context_.next_arg_id());
  }

  // 获取参数函数重载，根据 int 类型的 ID 获取参数
  FMT_CONSTEXPR format_arg get_arg(int arg_id) {
    parse_context_.check_arg_id(arg_id);
    return detail::get_arg(context_, arg_id);
  }

  // 获取参数函数重载，根据字符串视图类型的 ID 获取参数
  FMT_CONSTEXPR format_arg get_arg(basic_string_view<char_type> arg_id) {
    # 调用parse_context_对象的check_arg_id方法，检查arg_id参数
    parse_context_.check_arg_id(arg_id);
    # 返回detail命名空间中get_arg方法的结果，传入context_和arg_id参数
    return detail::get_arg(context_, arg_id);
  }

  # 声明ParseContext类型的引用parse_context_
  ParseContext& parse_context_;
  # 声明Context类型的引用context_
  Context& context_;
};
// 枚举类型，表示参数引用的类型
enum class arg_id_kind { none, index, name };

// 一个参数引用
template <typename Char> struct arg_ref {
  FMT_CONSTEXPR arg_ref() : kind(arg_id_kind::none), val() {}

  FMT_CONSTEXPR explicit arg_ref(int index)
      : kind(arg_id_kind::index), val(index) {}
  FMT_CONSTEXPR explicit arg_ref(basic_string_view<Char> name)
      : kind(arg_id_kind::name), val(name) {}

  FMT_CONSTEXPR arg_ref& operator=(int idx) {
    kind = arg_id_kind::index;
    val.index = idx;
    return *this;
  }

  arg_id_kind kind;
  union value {
    FMT_CONSTEXPR value(int id = 0) : index{id} {}
    FMT_CONSTEXPR value(basic_string_view<Char> n) : name(n) {}

    int index;
    basic_string_view<Char> name;
  } val;
};

// 在格式化时解析宽度和精度，而不是在解析时解析，以允许在不同参数集合下重用相同的解析规范（格式字符串的预编译）
template <typename Char>
struct dynamic_format_specs : basic_format_specs<Char> {
  arg_ref<Char> width_ref;
  arg_ref<Char> precision_ref;
};

// 格式规范处理程序，保存表示动态宽度和精度的参数引用，以便在格式化时解析
template <typename ParseContext>
class dynamic_specs_handler
    : public specs_setter<typename ParseContext::char_type> {
 public:
  using char_type = typename ParseContext::char_type;

  FMT_CONSTEXPR dynamic_specs_handler(dynamic_format_specs<char_type>& specs,
                                      ParseContext& ctx)
      : specs_setter<char_type>(specs), specs_(specs), context_(ctx) {}

  FMT_CONSTEXPR dynamic_specs_handler(const dynamic_specs_handler& other)
      : specs_setter<char_type>(other),
        specs_(other.specs_),
        context_(other.context_) {}

  template <typename Id> FMT_CONSTEXPR void on_dynamic_width(Id arg_id) {
    specs_.width_ref = make_arg_ref(arg_id);
  }

  template <typename Id> FMT_CONSTEXPR void on_dynamic_precision(Id arg_id) {
  // 设置精度参考为给定参数的引用
  specs_.precision_ref = make_arg_ref(arg_id);
}

FMT_CONSTEXPR void on_error(const char* message) {
  // 调用上下文对象的错误处理函数
  context_.on_error(message);
}

private:
using arg_ref_type = arg_ref<char_type>;

FMT_CONSTEXPR arg_ref_type make_arg_ref(int arg_id) {
  // 检查参数 ID 的有效性
  context_.check_arg_id(arg_id);
  // 返回参数的引用
  return arg_ref_type(arg_id);
}

FMT_CONSTEXPR arg_ref_type make_arg_ref(auto_id) {
  // 返回下一个参数的引用
  return arg_ref_type(context_.next_arg_id());
}

FMT_CONSTEXPR arg_ref_type make_arg_ref(basic_string_view<char_type> arg_id) {
  // 检查参数 ID 的有效性
  context_.check_arg_id(arg_id);
  // 创建参数 ID 对应的字符串视图
  basic_string_view<char_type> format_str(
      context_.begin(), to_unsigned(context_.end() - context_.begin()));
  // 返回参数的引用
  return arg_ref_type(arg_id);
}

// 动态格式规范的引用和解析上下文的引用
dynamic_format_specs<char_type>& specs_;
ParseContext& context_;
};
// 解析参数标识符，根据不同的标识符调用不同的处理函数
template <typename Char, typename IDHandler>
FMT_CONSTEXPR const Char* parse_arg_id(const Char* begin, const Char* end,
                                       IDHandler&& handler) {
  // 断言开始位置不等于结束位置
  FMT_ASSERT(begin != end, "");
  // 获取当前字符
  Char c = *begin;
  // 如果当前字符为 '}' 或者 ':'
  if (c == '}' || c == ':') {
    // 调用处理函数
    handler();
    return begin;
  }
  // 如果当前字符为数字
  if (c >= '0' && c <= '9') {
    int index = 0;
    // 如果当前字符不为 '0'，则解析非负整数
    if (c != '0')
      index = parse_nonnegative_int(begin, end, handler);
    else
      ++begin;
    // 如果已经到达结束位置或者下一个字符不是 '}' 或者 ':'
    if (begin == end || (*begin != '}' && *begin != ':'))
      // 报错
      handler.on_error("invalid format string");
    else
      // 调用处理函数
      handler(index);
    return begin;
  }
  // 如果当前字符不是有效的名称起始字符
  if (!is_name_start(c)) {
    // 报错
    handler.on_error("invalid format string");
    return begin;
  }
  // 寻找名称结束位置
  auto it = begin;
  do {
    ++it;
  } while (it != end && (is_name_start(c = *it) || ('0' <= c && c <= '9')));
  // 调用处理函数
  handler(basic_string_view<Char>(begin, to_unsigned(it - begin)));
  return it;
}

// 适配 SpecHandler 到 IDHandler API 以处理动态宽度
template <typename SpecHandler, typename Char> struct width_adapter {
  explicit FMT_CONSTEXPR width_adapter(SpecHandler& h) : handler(h) {}

  FMT_CONSTEXPR void operator()() { handler.on_dynamic_width(auto_id()); }
  FMT_CONSTEXPR void operator()(int id) { handler.on_dynamic_width(id); }
  FMT_CONSTEXPR void operator()(basic_string_view<Char> id) {
    handler.on_dynamic_width(id);
  }

  FMT_CONSTEXPR void on_error(const char* message) {
    handler.on_error(message);
  }

  SpecHandler& handler;
};

// 适配 SpecHandler 到 IDHandler API 以处理动态精度
template <typename SpecHandler, typename Char> struct precision_adapter {
  explicit FMT_CONSTEXPR precision_adapter(SpecHandler& h) : handler(h) {}

  FMT_CONSTEXPR void operator()() { handler.on_dynamic_precision(auto_id()); }
  FMT_CONSTEXPR void operator()(int id) { handler.on_dynamic_precision(id); }
  FMT_CONSTEXPR void operator()(basic_string_view<Char> id) {
    handler.on_dynamic_precision(id);
  }

  FMT_CONSTEXPR void on_error(const char* message) {
  # 调用 handler 对象的 on_error 方法，并传入 message 参数
  handler.on_error(message);
}

# 声明一个引用类型的 handler 对象，类型为 SpecHandler
SpecHandler& handler;
// 用于查找下一个代码点的函数模板，根据不同的字符类型进行处理
template <typename Char>
FMT_CONSTEXPR const Char* next_code_point(const Char* begin, const Char* end) {
  // 如果字符类型不是1字节，或者第一个字符的最高位不是1，则直接返回下一个字符的指针
  if (const_check(sizeof(Char) != 1) || (*begin & 0x80) == 0) return begin + 1;
  // 否则，循环查找下一个代码点的起始位置
  do {
    ++begin;
  } while (begin != end && (*begin & 0xc0) == 0x80);
  return begin;
}

// 解析填充字符和对齐方式的函数模板
template <typename Char, typename Handler>
FMT_CONSTEXPR const Char* parse_align(const Char* begin, const Char* end,
                                      Handler&& handler) {
  // 断言开始位置不等于结束位置
  FMT_ASSERT(begin != end, "");
  auto align = align::none;
  auto p = next_code_point(begin, end);
  // 如果下一个代码点的位置等于结束位置，则将 p 重新指向 begin
  if (p == end) p = begin;
  // 循环解析填充字符和对齐方式
  for (;;) {
    switch (static_cast<int>(*p)) {
    case '<':
      align = align::left;
      break;
    case '>':
      align = align::right;
      break;
#if FMT_DEPRECATED_NUMERIC_ALIGN
    case '=':
      align = align::numeric;
      break;
#endif
    case '^':
      align = align::center;
      break;
    }
    // 如果找到了对齐方式
    if (align != align::none) {
      // 如果 p 不等于 begin，则表示存在填充字符
      if (p != begin) {
        auto c = *begin;
        // 如果填充字符是 '{'，则返回错误
        if (c == '{')
          return handler.on_error("invalid fill character '{'"), begin;
        // 调用处理器的 on_fill 方法，传入填充字符和长度
        handler.on_fill(basic_string_view<Char>(begin, to_unsigned(p - begin)));
        begin = p + 1;
      } else
        ++begin;
      // 调用处理器的 on_align 方法，传入对齐方式
      handler.on_align(align);
      break;
    } else if (p == begin) {
      break;
    }
    p = begin;
  }
  return begin;
}

// 解析宽度的函数模板
template <typename Char, typename Handler>
FMT_CONSTEXPR const Char* parse_width(const Char* begin, const Char* end,
                                      Handler&& handler) {
  // 断言开始位置不等于结束位置
  FMT_ASSERT(begin != end, "");
  // 如果第一个字符是数字，则解析宽度
  if ('0' <= *begin && *begin <= '9') {
    handler.on_width(parse_nonnegative_int(begin, end, handler));
  } else if (*begin == '{') {
    ++begin;
    // 如果开始位置不等于结束位置，则解析参数 ID
    if (begin != end)
      begin = parse_arg_id(begin, end, width_adapter<Handler, Char>(handler));
    // 如果开始位置等于结束位置，或者下一个字符不是 '}'，则返回错误
    if (begin == end || *begin != '}')
      return handler.on_error("invalid format string"), begin;
    ++begin;
  }
  return begin;
}
// 解析精度信息，更新处理器的状态，并返回更新后的迭代器位置
template <typename Char, typename Handler>
FMT_CONSTEXPR const Char* parse_precision(const Char* begin, const Char* end,
                                          Handler&& handler) {
  // 增加迭代器位置，跳过精度标识符 '.'
  ++begin;
  // 获取下一个字符
  auto c = begin != end ? *begin : Char();
  // 如果下一个字符是数字，则解析非负整数作为精度
  if ('0' <= c && c <= '9') {
    handler.on_precision(parse_nonnegative_int(begin, end, handler));
  } else if (c == '{') {
    // 如果下一个字符是 '{'，则解析参数 ID 作为精度
    ++begin;
    if (begin != end) {
      begin =
          parse_arg_id(begin, end, precision_adapter<Handler, Char>(handler));
    }
    // 如果解析失败，返回错误信息
    if (begin == end || *begin++ != '}')
      return handler.on_error("invalid format string"), begin;
  } else {
    // 如果没有精度标识符，返回错误信息
    return handler.on_error("missing precision specifier"), begin;
  }
  // 结束精度解析，更新处理器状态
  handler.end_precision();
  // 返回更新后的迭代器位置
  return begin;
}

// 解析标准格式规范，并将解析的组件通知给处理器
template <typename Char, typename SpecHandler>
FMT_CONSTEXPR const Char* parse_format_specs(const Char* begin, const Char* end,
                                             SpecHandler&& handler) {
  // 如果开始位置等于结束位置，或者开始位置的字符为 '}'，则直接返回开始位置
  if (begin == end || *begin == '}') return begin;

  // 解析对齐规则
  begin = parse_align(begin, end, handler);
  if (begin == end) return begin;

  // 解析符号
  switch (static_cast<char>(*begin)) {
  case '+':
    handler.on_plus();
    ++begin;
    break;
  case '-':
    handler.on_minus();
    ++begin;
    break;
  case ' ':
    handler.on_space();
    ++begin;
    break;
  }
  if (begin == end) return begin;

  // 如果下一个字符为 '#'，则通知处理器
  if (*begin == '#') {
    handler.on_hash();
    if (++begin == end) return begin;
  }

  // 解析零标识符
  if (*begin == '0') {
    handler.on_zero();
    if (++begin == end) return begin;
  }

  // 解析宽度
  begin = parse_width(begin, end, handler);
  if (begin == end) return begin;

  // 解析精度
  if (*begin == '.') {
    begin = parse_precision(begin, end, handler);
  }

  // 解析类型
  if (begin != end && *begin != '}') handler.on_type(*begin++);
  // 返回更新后的迭代器位置
  return begin;
}
// 定义一个模板函数，用于在指定范围内查找特定值，并返回是否找到以及找到的位置
template <bool IS_CONSTEXPR, typename T, typename Ptr = const T*>
FMT_CONSTEXPR bool find(Ptr first, Ptr last, T value, Ptr& out) {
  // 遍历指定范围内的元素
  for (out = first; out != last; ++out) {
    // 如果找到特定值，则返回 true
    if (*out == value) return true;
  }
  // 如果未找到特定值，则返回 false
  return false;
}

// 对于特定的类型和值，提供特化的查找函数
template <>
inline bool find<false, char>(const char* first, const char* last, char value,
                              const char*& out) {
  // 使用 memchr 函数在指定范围内查找特定值
  out = static_cast<const char*>(
      std::memchr(first, value, detail::to_unsigned(last - first)));
  // 返回是否找到特定值
  return out != nullptr;
}

// 定义一个结构体，用于适配处理程序和字符类型
template <typename Handler, typename Char> struct id_adapter {
  Handler& handler;
  int arg_id;

  // 重载函数调用操作符，根据参数类型调用不同的处理程序
  FMT_CONSTEXPR void operator()() { arg_id = handler.on_arg_id(); }
  FMT_CONSTEXPR void operator()(int id) { arg_id = handler.on_arg_id(id); }
  FMT_CONSTEXPR void operator()(basic_string_view<Char> id) {
    arg_id = handler.on_arg_id(id);
  }
  FMT_CONSTEXPR void on_error(const char* message) {
    handler.on_error(message);
  }
};

// 解析替换字段的函数模板，根据不同情况调用处理程序
template <typename Char, typename Handler>
FMT_CONSTEXPR const Char* parse_replacement_field(const Char* begin,
                                                  const Char* end,
                                                  Handler&& handler) {
  // 移动到下一个字符
  ++begin;
  // 如果已经到达末尾，则返回错误
  if (begin == end) return handler.on_error("invalid format string"), end;
  // 如果下一个字符是 '}'，则调用处理程序的 on_replacement_field 函数
  if (static_cast<char>(*begin) == '}') {
    handler.on_replacement_field(handler.on_arg_id(), begin);
  } else if (*begin == '{') {
    handler.on_text(begin, begin + 1);
  } else {
    // 创建适配器，根据情况调用不同的处理程序
    auto adapter = id_adapter<Handler, Char>{handler, 0};
    begin = parse_arg_id(begin, end, adapter);
    Char c = begin != end ? *begin : Char();
    if (c == '}') {
      handler.on_replacement_field(adapter.arg_id, begin);
    } else if (c == ':') {
      begin = handler.on_format_specs(adapter.arg_id, begin + 1, end);
      if (begin == end || *begin != '}')
        return handler.on_error("unknown format specifier"), end;
    } else {
      return handler.on_error("missing '}' in format string"), end;
    }
  }
}
    }  # 结束内层循环的代码块
  }  # 结束外层循环的代码块
  return begin + 1;  # 返回 begin 变量的值加 1
// 解析格式字符串，根据 IS_CONSTEXPR 的值选择不同的实现方式
template <bool IS_CONSTEXPR, typename Char, typename Handler>
FMT_CONSTEXPR_DECL FMT_INLINE void parse_format_string(
    basic_string_view<Char> format_str, Handler&& handler) {
  // 获取格式字符串的起始和结束位置
  auto begin = format_str.data();
  auto end = begin + format_str.size();
  // 如果字符串长度小于32，则使用简单循环而不是 memchr
  if (end - begin < 32) {
    const Char* p = begin;
    // 遍历字符串，处理 '{' 和 '}' 符号
    while (p != end) {
      auto c = *p++;
      if (c == '{') {
        handler.on_text(begin, p - 1);
        begin = p = parse_replacement_field(p - 1, end, handler);
      } else if (c == '}') {
        if (p == end || *p != '}')
          return handler.on_error("unmatched '}' in format string");
        handler.on_text(begin, p);
        begin = ++p;
      }
    }
    handler.on_text(begin, end);
    return;
  }
  // 定义一个内部结构体 writer，用于处理大字符串的情况
  struct writer {
    FMT_CONSTEXPR void operator()(const Char* pbegin, const Char* pend) {
      if (pbegin == pend) return;
      for (;;) {
        const Char* p = nullptr;
        // 查找 '}' 符号
        if (!find<IS_CONSTEXPR>(pbegin, pend, '}', p))
          return handler_.on_text(pbegin, pend);
        ++p;
        if (p == pend || *p != '}')
          return handler_.on_error("unmatched '}' in format string");
        handler_.on_text(pbegin, p);
        pbegin = p + 1;
      }
    }
    Handler& handler_;
  } write{handler};
  // 遍历字符串，处理 '{' 和 '}' 符号
  while (begin != end) {
    const Char* p = begin;
    if (*begin != '{' && !find<IS_CONSTEXPR>(begin + 1, end, '{', p))
      return write(begin, end);
    write(begin, p);
    begin = parse_replacement_field(p, end, handler);
  }
}

// 解析格式规范
template <typename T, typename ParseContext>
FMT_CONSTEXPR const typename ParseContext::char_type* parse_format_specs(
    // 定义一个引用类型的参数 ctx，类型为 ParseContext
    ParseContext& ctx) {
  // 使用别名 char_type 表示 ParseContext 中的字符类型
  using char_type = typename ParseContext::char_type;
  // 使用别名 context 表示 buffer_context<char_type>
  using context = buffer_context<char_type>;
  // 使用别名 mapped_type 表示条件类型，如果 detail::mapped_type_constant<T, context>::value 不等于 type::custom_type，则为 decltype(arg_mapper<context>().map(std::declval<T>()))，否则为 T
  using mapped_type =
      conditional_t<detail::mapped_type_constant<T, context>::value !=
                        type::custom_type,
                    decltype(arg_mapper<context>().map(std::declval<T>())), T>;
  // 定义一个自动类型的变量 f，如果 mapped_type 有 formatter 并且 context 中有 formatter，则为 formatter<mapped_type, char_type>，否则为 detail::fallback_formatter<T, char_type>
  auto f = conditional_t<has_formatter<mapped_type, context>::value,
                         formatter<mapped_type, char_type>,
                         detail::fallback_formatter<T, char_type>>();
  // 调用 f 的 parse 方法，解析参数 ctx
  return f.parse(ctx);
// 定义一个模板结构体，用于处理格式化输出的参数
template <typename ArgFormatter, typename Char, typename Context>
struct format_handler : detail::error_handler {
  // 解析上下文对象
  basic_format_parse_context<Char> parse_context;
  // 格式化输出的上下文对象
  Context context;

  // 构造函数，初始化解析上下文和格式化输出上下文
  format_handler(typename ArgFormatter::iterator out,
                 basic_string_view<Char> str,
                 basic_format_args<Context> format_args, detail::locale_ref loc)
      : parse_context(str), context(out, format_args, loc) {}

  // 处理文本内容
  void on_text(const Char* begin, const Char* end) {
    // 计算文本内容的大小
    auto size = to_unsigned(end - begin);
    // 获取格式化输出的迭代器
    auto out = context.out();
    // 预留足够的空间
    auto&& it = reserve(out, size);
    // 将文本内容复制到输出迭代器
    it = std::copy_n(begin, size, it);
    // 更新格式化输出的位置
    context.advance_to(out);
  }

  // 处理参数的索引
  int on_arg_id() { return parse_context.next_arg_id(); }
  // 处理指定的参数索引
  int on_arg_id(int id) { return parse_context.check_arg_id(id), id; }
  // 处理参数的索引
  int on_arg_id(basic_string_view<Char> id) {
    // 获取参数的索引
    int arg_id = context.arg_id(id);
    // 如果参数索引小于0，则表示参数未找到，报错
    if (arg_id < 0) on_error("argument not found");
    return arg_id;
  }

  // 处理替换字段
  FMT_INLINE void on_replacement_field(int id, const Char*) {
    // 获取指定索引的参数
    auto arg = get_arg(context, id);
    // 更新格式化输出的位置
    context.advance_to(visit_format_arg(
        default_arg_formatter<typename ArgFormatter::iterator, Char>{
            context.out(), context.args(), context.locale()},
        arg));
  }

  // 处理格式规范
  const Char* on_format_specs(int id, const Char* begin, const Char* end) {
    // 更新解析上下文的位置
    advance_to(parse_context, begin);
    // 获取指定索引的参数
    auto arg = get_arg(context, id);
    // 创建自定义格式化对象
    custom_formatter<Context> f(parse_context, context);
    // 如果成功格式化，则返回解析开始位置
    if (visit_format_arg(f, arg)) return parse_context.begin();
    // 创建基本格式规范对象
    basic_format_specs<Char> specs;
    // 创建格式规范处理对象
    using parse_context_t = basic_format_parse_context<Char>;
    specs_checker<specs_handler<parse_context_t, Context>> handler(
        specs_handler<parse_context_t, Context>(specs, parse_context, context),
        arg.type());
    // 解析格式规范
    begin = parse_format_specs(begin, end, handler);
    // 如果解析失败或者结束位置不是 '}'，则报错
    if (begin == end || *begin != '}') on_error("missing '}' in format string");
    // 更新解析上下文的位置
    advance_to(parse_context, begin);
    # 将上下文推进到指定位置，使用格式化参数和参数值进行格式化
    context.advance_to(
        visit_format_arg(ArgFormatter(context, &parse_context, &specs), arg));
    # 返回起始位置
    return begin;
  }
};

// 带有额外参数 id 检查的解析上下文。仅在编译时使用，因为在运行时添加检查会引入重大开销，
// 并且在检索参数时会检查参数 id，因此会是多余的。
template <typename Char, typename ErrorHandler = error_handler>
class compile_parse_context
    : public basic_format_parse_context<Char, ErrorHandler> {
 private:
  int num_args_;
  using base = basic_format_parse_context<Char, ErrorHandler>;

 public:
  explicit FMT_CONSTEXPR compile_parse_context(
      basic_string_view<Char> format_str, int num_args = max_value<int>(),
      ErrorHandler eh = {})
      : base(format_str, eh), num_args_(num_args) {}

  FMT_CONSTEXPR int next_arg_id() {
    int id = base::next_arg_id();
    if (id >= num_args_) this->on_error("argument not found");
    return id;
  }

  FMT_CONSTEXPR void check_arg_id(int id) {
    base::check_arg_id(id);
    if (id >= num_args_) this->on_error("argument not found");
  }
  using base::check_arg_id;
};

template <typename Char, typename ErrorHandler, typename... Args>
class format_string_checker {
 public:
  explicit FMT_CONSTEXPR format_string_checker(
      basic_string_view<Char> format_str, ErrorHandler eh)
      : context_(format_str, num_args, eh),
        parse_funcs_{&parse_format_specs<Args, parse_context_type>...} {}

  FMT_CONSTEXPR void on_text(const Char*, const Char*) {}

  FMT_CONSTEXPR int on_arg_id() { return context_.next_arg_id(); }
  FMT_CONSTEXPR int on_arg_id(int id) { return context_.check_arg_id(id), id; }
  FMT_CONSTEXPR int on_arg_id(basic_string_view<Char>) {
    on_error("compile-time checks don't support named arguments");
    return 0;
  }

  FMT_CONSTEXPR void on_replacement_field(int, const Char*) {}

  FMT_CONSTEXPR const Char* on_format_specs(int id, const Char* begin,
                                            const Char*) {
    advance_to(context_, begin);
    // 如果 id 小于 num_args，则调用相应的解析函数，否则返回 begin
    return id < num_args ? parse_funcs_[id](context_) : begin;
  }

  // 当出现错误时，调用上下文对象的 on_error 方法
  FMT_CONSTEXPR void on_error(const char* message) {
    context_.on_error(message);
  }

 private:
  // 定义解析上下文类型
  using parse_context_type = compile_parse_context<Char, ErrorHandler>;
  // 计算模板参数包中的参数个数
  enum { num_args = sizeof...(Args) };

  // 格式说明符解析函数类型
  using parse_func = const Char* (*)(parse_context_type&);

  // 解析上下文对象
  parse_context_type context_;
  // 解析函数指针数组，数组大小为 num_args 或 1（如果 num_args 为 0）
  parse_func parse_funcs_[num_args > 0 ? num_args : 1];
// 结束了一个代码块
};

// 将字符串文字转换为 basic_string_view。
template <typename Char, size_t N>
FMT_CONSTEXPR basic_string_view<Char> compile_string_to_view(
    const Char (&s)[N]) {
  // 如果需要，移除末尾的空字符。如果使用原始字符数组（即未定义为字符串），则不会存在。
  return {s,
          N - ((std::char_traits<Char>::to_int_type(s[N - 1]) == 0) ? 1 : 0)};
}

// 将 string_view 转换为 basic_string_view。
template <typename Char>
FMT_CONSTEXPR basic_string_view<Char> compile_string_to_view(
    const std_string_view<Char>& s) {
  return {s.data(), s.size()};
}

// 定义 FMT_STRING_IMPL 宏
#define FMT_STRING_IMPL(s, base)                                  \
  [] {                                                            \
    /* 使用类似宏的名称以避免阴影警告。 */      
    struct FMT_COMPILE_STRING : base {                            \
      using char_type = fmt::remove_cvref_t<decltype(s[0])>;      \
      FMT_MAYBE_UNUSED FMT_CONSTEXPR                              \
      operator fmt::basic_string_view<char_type>() const {        \
        return fmt::detail::compile_string_to_view<char_type>(s); \
      }                                                           \
    };                                                            \
    return FMT_COMPILE_STRING();                                  \
  }()

/**
  \rst
  从字符串文字 *s* 构造一个编译时格式字符串。

  **Example**::

    // A compile-time error because 'd' is an invalid specifier for strings.
    std::string s = fmt::format(FMT_STRING("{:d}"), "foo");
  \endrst
 */
// 定义 FMT_STRING 宏
#define FMT_STRING(s) FMT_STRING_IMPL(s, fmt::compile_string)

// 模板函数，接受参数包 Args 和 S，当 S 是编译时字符串时启用
template <typename... Args, typename S,
          enable_if_t<(is_compile_string<S>::value), int>>
// 检查格式字符串是否合法
void check_format_string(S format_str) {
  // 将格式字符串转换为字符串视图
  FMT_CONSTEXPR_DECL auto s = to_string_view(format_str);
  // 使用指定的错误处理程序和参数类型检查格式字符串
  using checker = format_string_checker<typename S::char_type, error_handler, remove_cvref_t<Args>...>;
  // 解析格式字符串并检查是否合法
  FMT_CONSTEXPR_DECL bool invalid_format =
      (parse_format_string<true>(s, checker(s, {})), true);
  // 防止编译器警告
  (void)invalid_format;
}

// 处理动态规格
template <template <typename> class Handler, typename Context>
void handle_dynamic_spec(int& value, arg_ref<typename Context::char_type> ref,
                         Context& ctx) {
  // 根据引用类型处理动态规格
  switch (ref.kind) {
  case arg_id_kind::none:
    break;
  case arg_id_kind::index:
    // 获取索引对应的动态规格值
    value = detail::get_dynamic_spec<Handler>(ctx.arg(ref.val.index),
                                              ctx.error_handler());
    break;
  case arg_id_kind::name:
    // 获取名称对应的动态规格值
    value = detail::get_dynamic_spec<Handler>(ctx.arg(ref.val.name),
                                              ctx.error_handler());
    break;
  }
}

// 格式化错误码
using format_func = void (*)(detail::buffer<char>&, int, string_view);

FMT_API void format_error_code(buffer<char>& out, int error_code,
                               string_view message) FMT_NOEXCEPT;

FMT_API void report_error(format_func func, int error_code,
                          string_view message) FMT_NOEXCEPT;

/** 默认的参数格式化器 */
template <typename OutputIt, typename Char>
class arg_formatter : public arg_formatter_base<OutputIt, Char> {
 private:
  using char_type = Char;
  using base = arg_formatter_base<OutputIt, Char>;
  using context_type = basic_format_context<OutputIt, Char>;

  context_type& ctx_;
  basic_format_parse_context<char_type>* parse_ctx_;
  const Char* ptr_;

 public:
  using iterator = typename base::iterator;
  using format_specs = typename base::format_specs;

  /**
    \rst
    构造参数格式化器对象。
    *ctx* 是格式化上下文的引用，
    *specs* 包含标准参数类型的格式说明信息。
  ```
  \endrst
 */
// 使用给定的上下文、解析上下文、格式规范和指针创建 arg_formatter 对象
explicit arg_formatter(
    context_type& ctx,
    basic_format_parse_context<char_type>* parse_ctx = nullptr,
    format_specs* specs = nullptr, const Char* ptr = nullptr)
    : base(ctx.out(), specs, ctx.locale()),
      ctx_(ctx),
      parse_ctx_(parse_ctx),
      ptr_(ptr) {}

// 使用基类的 operator() 方法
using base::operator();

/** 格式化用户定义类型的参数 */
iterator operator()(typename basic_format_arg<context_type>::handle handle) {
  // 如果指针存在，则将解析上下文移动到指定位置
  if (ptr_) advance_to(*parse_ctx_, ptr_);
  // 调用 handle 对象的 format 方法，格式化参数
  handle.format(*parse_ctx_, ctx_);
  // 返回输出迭代器
  return ctx_.out();
}
// 结束 detail 命名空间
};
// 结束 namespace detail

// 使用模板别名定义 arg_formatter
template <typename OutputIt, typename Char>
using arg_formatter FMT_DEPRECATED_ALIAS =
    detail::arg_formatter<OutputIt, Char>;

/**
 表示操作系统或语言运行时返回的错误，例如文件打开错误。
*/
// 使用 FMT_CLASS_API 宏修饰，表示该类是公共 API
FMT_CLASS_API
class FMT_API system_error : public std::runtime_error {
 private:
  // 初始化函数，用于设置错误码和格式化错误信息
  void init(int err_code, string_view format_str, format_args args);

 protected:
  // 错误码
  int error_code_;

  // 默认构造函数，初始化错误码为 0
  system_error() : std::runtime_error(""), error_code_(0) {}

 public:
  /**
   \rst
   使用 `fmt::format_system_error` 格式化描述信息构造 :class:`fmt::system_error` 对象。
   *message* 和传入构造函数的额外参数会被类似于 `fmt::format` 的方式格式化。

   **示例**::

     // 这会抛出一个带有描述信息的 system_error
     //   例如：cannot open file 'madeup': No such file or directory
     // 系统消息可能会有所不同。
     const char *filename = "madeup";
     std::FILE *file = std::fopen(filename, "r");
     if (!file)
       throw fmt::system_error(errno, "cannot open file '{}'", filename);
   \endrst
  */
  template <typename... Args>
  system_error(int error_code, string_view message, const Args&... args)
      : std::runtime_error("") {
    init(error_code, message, make_format_args(args...));
  }
  // 复制构造函数
  system_error(const system_error&) = default;
  // 复制赋值运算符
  system_error& operator=(const system_error&) = default;
  // 移动构造函数
  system_error(system_error&&) = default;
  // 移动赋值运算符
  system_error& operator=(system_error&&) = default;
  // 析构函数
  ~system_error() FMT_NOEXCEPT FMT_OVERRIDE;

  // 返回错误码
  int error_code() const { return error_code_; }
};
/**
  \rst
  Formats an error returned by an operating system or a language runtime,
  for example a file opening error, and writes it to *out* in the following
  form:

  .. parsed-literal::
     *<message>*: *<system-message>*

  where *<message>* is the passed message and *<system-message>* is
  the system message corresponding to the error code.
  *error_code* is a system error code as given by ``errno``.
  If *error_code* is not a valid error code such as -1, the system message
  may look like "Unknown error -1" and is platform-dependent.
  \endrst
 */
FMT_API void format_system_error(detail::buffer<char>& out, int error_code,
                                 string_view message) FMT_NOEXCEPT;

// Reports a system error without throwing an exception.
// Can be used to report errors from destructors.
FMT_API void report_system_error(int error_code,
                                 string_view message) FMT_NOEXCEPT;

/** Fast integer formatter. */
class format_int {
 private:
  // Buffer should be large enough to hold all digits (digits10 + 1),
  // a sign and a null character.
  enum { buffer_size = std::numeric_limits<unsigned long long>::digits10 + 3 };
  mutable char buffer_[buffer_size];
  char* str_;

  template <typename UInt> char* format_unsigned(UInt value) {
    auto n = static_cast<detail::uint32_or_64_or_128_t<UInt>>(value);
    return detail::format_decimal(buffer_, n, buffer_size - 1).begin;
  }

  template <typename Int> char* format_signed(Int value) {
    auto abs_value = static_cast<detail::uint32_or_64_or_128_t<Int>>(value);
    bool negative = value < 0;
    if (negative) abs_value = 0 - abs_value;
    auto begin = format_unsigned(abs_value);
    if (negative) *--begin = '-';
  }
  // 返回 begin 变量的值
  return begin;
}

public:
explicit format_int(int value) : str_(format_signed(value)) {}
explicit format_int(long value) : str_(format_signed(value)) {}
explicit format_int(long long value) : str_(format_signed(value)) {}
explicit format_int(unsigned value) : str_(format_unsigned(value)) {}
explicit format_int(unsigned long value) : str_(format_unsigned(value)) {}
explicit format_int(unsigned long long value)
    : str_(format_unsigned(value)) {}

/** 返回写入输出缓冲区的字符数。 */
size_t size() const {
  return detail::to_unsigned(buffer_ - str_ + buffer_size - 1);
}

/**
  返回指向输出缓冲区内容的指针。不附加终止的空字符。
 */
const char* data() const { return str_; }

/**
  返回指向附加终止空字符的输出缓冲区内容的指针。
 */
const char* c_str() const {
  buffer_[buffer_size - 1] = '\0';
  return str_;
}

/**
  \rst
  将输出缓冲区的内容作为 ``std::string`` 返回。
  \endrst
 */
std::string str() const { return std::string(str_, size()); }
};

// 为与 detail::type 常量对应的核心类型的格式化器特化
template <typename T, typename Char>
struct formatter<T, Char,
                 enable_if_t<detail::type_constant<T, Char>::value !=
                             detail::type::custom_type>> {
  FMT_CONSTEXPR formatter() = default;

  // 解析格式说明符，停止在范围结束或终止的 '}'
  template <typename ParseContext>
  FMT_CONSTEXPR auto parse(ParseContext& ctx) -> decltype(ctx.begin()) {
    using handler_type = detail::dynamic_specs_handler<ParseContext>;
    auto type = detail::type_constant<T, Char>::value;
    detail::specs_checker<handler_type> handler(handler_type(specs_, ctx),
                                                type);
    auto it = parse_format_specs(ctx.begin(), ctx.end(), handler);
    auto eh = ctx.error_handler();
    switch (type) {
    case detail::type::none_type:
      FMT_ASSERT(false, "invalid argument type");
      break;
    case detail::type::int_type:
    case detail::type::uint_type:
    case detail::type::long_long_type:
    case detail::type::ulong_long_type:
    case detail::type::int128_type:
    case detail::type::uint128_type:
    case detail::type::bool_type:
      handle_int_type_spec(specs_.type,
                           detail::int_type_checker<decltype(eh)>(eh));
      break;
    case detail::type::char_type:
      handle_char_specs(
          &specs_, detail::char_specs_checker<decltype(eh)>(specs_.type, eh));
      break;
    case detail::type::float_type:
      if (detail::const_check(FMT_USE_FLOAT))
        detail::parse_float_type_spec(specs_, eh);
      else
        FMT_ASSERT(false, "float support disabled");
      break;
    case detail::type::double_type:
      if (detail::const_check(FMT_USE_DOUBLE))
        detail::parse_float_type_spec(specs_, eh);
      else
        FMT_ASSERT(false, "double support disabled");
      break;
    // 如果值的类型是 long double，则根据 FMT_USE_LONG_DOUBLE 的设置来解析浮点数类型的规范
    case detail::type::long_double_type:
      if (detail::const_check(FMT_USE_LONG_DOUBLE))
        detail::parse_float_type_spec(specs_, eh);
      else
        FMT_ASSERT(false, "long double support disabled");
      break;
    // 如果值的类型是 C 字符串类型，则处理 C 字符串类型的规范
    case detail::type::cstring_type:
      detail::handle_cstring_type_spec(
          specs_.type, detail::cstring_type_checker<decltype(eh)>(eh));
      break;
    // 如果值的类型是字符串类型，则检查字符串类型的规范
    case detail::type::string_type:
      detail::check_string_type_spec(specs_.type, eh);
      break;
    // 如果值的类型是指针类型，则检查指针类型的规范
    case detail::type::pointer_type:
      detail::check_pointer_type_spec(specs_.type, eh);
      break;
    // 如果值的类型是自定义类型，则在格式化器特化的解析函数中检查自定义格式规范
    case detail::type::custom_type:
      // Custom format specifiers should be checked in parse functions of
      // formatter specializations.
      break;
    }
    // 返回迭代器
    return it;
  }

  // 格式化值并将结果输出到上下文中
  template <typename FormatContext>
  auto format(const T& val, FormatContext& ctx) -> decltype(ctx.out()) {
    // 处理动态宽度规范
    detail::handle_dynamic_spec<detail::width_checker>(specs_.width,
                                                       specs_.width_ref, ctx);
    // 处理动态精度规范
    detail::handle_dynamic_spec<detail::precision_checker>(
        specs_.precision, specs_.precision_ref, ctx);
    // 使用 arg_formatter 访问格式参数并返回结果
    using af = detail::arg_formatter<typename FormatContext::iterator,
                                     typename FormatContext::char_type>;
    return visit_format_arg(af(ctx, nullptr, &specs_),
                            detail::make_arg<FormatContext>(val));
  }

 private:
  // 存储动态格式规范
  detail::dynamic_format_specs<Char> specs_;
};

// 定义宏，用于将特定类型格式化为另一种类型
#define FMT_FORMAT_AS(Type, Base)                                             \
  template <typename Char>                                                    \
  struct formatter<Type, Char> : formatter<Base, Char> {                      \
    template <typename FormatContext>                                         \
    auto format(Type const& val, FormatContext& ctx) -> decltype(ctx.out()) { \
      return formatter<Base, Char>::format(val, ctx);                         \
    }                                                                         \
  }

// 将 signed char 格式化为 int
FMT_FORMAT_AS(signed char, int);
// 将 unsigned char 格式化为 unsigned
FMT_FORMAT_AS(unsigned char, unsigned);
// 将 short 格式化为 int
FMT_FORMAT_AS(short, int);
// 将 unsigned short 格式化为 unsigned
FMT_FORMAT_AS(unsigned short, unsigned);
// 将 long 格式化为 long long
FMT_FORMAT_AS(long, long long);
// 将 unsigned long 格式化为 unsigned long long
FMT_FORMAT_AS(unsigned long, unsigned long long);
// 将 Char* 格式化为 const Char*
FMT_FORMAT_AS(Char*, const Char*);
// 将 std::basic_string<Char> 格式化为 basic_string_view<Char>
FMT_FORMAT_AS(std::basic_string<Char>, basic_string_view<Char>);
// 将 std::nullptr_t 格式化为 const void*
FMT_FORMAT_AS(std::nullptr_t, const void*);
// 将 detail::std_string_view<Char> 格式化为 basic_string_view<Char>
FMT_FORMAT_AS(detail::std_string_view<Char>, basic_string_view<Char>);

// 用于格式化 void* 类型
template <typename Char>
struct formatter<void*, Char> : formatter<const void*, Char> {
  template <typename FormatContext>
  auto format(void* val, FormatContext& ctx) -> decltype(ctx.out()) {
    return formatter<const void*, Char>::format(val, ctx);
  }
};

// 用于格式化 Char[N] 类型
template <typename Char, size_t N>
struct formatter<Char[N], Char> : formatter<basic_string_view<Char>, Char> {
  template <typename FormatContext>
  auto format(const Char* val, FormatContext& ctx) -> decltype(ctx.out()) {
    return formatter<basic_string_view<Char>, Char>::format(val, ctx);
  }
};

// 用于运行时类型的格式化，例如 variant 类型
// 使用示例：
//   using variant = std::variant<int, std::string>;
//   template <>
//   struct formatter<variant>: dynamic_formatter<> {
//     auto format(const variant& v, format_context& ctx) {
//       return visit([&](const auto& val) {
//           return dynamic_formatter<>::format(val, ctx);
//       }, v);
//     }
//   };
template <typename Char = char> class dynamic_formatter {
 private:
  struct null_handler : detail::error_handler {
    void on_align(align_t) {}
    void on_plus() {}
    void on_minus() {}
    void on_space() {}
    void on_hash() {}
  };

 public:
  template <typename ParseContext>
  auto parse(ParseContext& ctx) -> decltype(ctx.begin()) {
    format_str_ = ctx.begin();
    // 在格式化时检查参数类型已知的情况下进行延迟检查
    detail::dynamic_specs_handler<ParseContext> handler(specs_, ctx);
    return parse_format_specs(ctx.begin(), ctx.end(), handler);
  }

  template <typename T, typename FormatContext>
  auto format(const T& val, FormatContext& ctx) -> decltype(ctx.out()) {
    handle_specs(ctx);
    detail::specs_checker<null_handler> checker(
        null_handler(), detail::mapped_type_constant<T, FormatContext>::value);
    checker.on_align(specs_.align);
    switch (specs_.sign) {
    case sign::none:
      break;
    case sign::plus:
      checker.on_plus();
      break;
    case sign::minus:
      checker.on_minus();
      break;
    case sign::space:
      checker.on_space();
      break;
    }
    if (specs_.alt) checker.on_hash();
    if (specs_.precision >= 0) checker.end_precision();
    using af = detail::arg_formatter<typename FormatContext::iterator,
                                     typename FormatContext::char_type>;
    visit_format_arg(af(ctx, nullptr, &specs_),
                     detail::make_arg<FormatContext>(val));
    return ctx.out();
  }

 private:
  template <typename Context> void handle_specs(Context& ctx) {
    detail::handle_dynamic_spec<detail::width_checker>(specs_.width,
                                                       specs_.width_ref, ctx);
    detail::handle_dynamic_spec<detail::precision_checker>(
        specs_.precision, specs_.precision_ref, ctx);
  }

  detail::dynamic_format_specs<Char> specs_;
  const Char* format_str_;
};

template <typename Char, typename ErrorHandler>
# 定义一个常量表达式函数，用于将解析上下文中的迭代器前进到指定位置
FMT_CONSTEXPR void advance_to(
    basic_format_parse_context<Char, ErrorHandler>& ctx, const Char* p) {
  ctx.advance_to(ctx.begin() + (p - &*ctx.begin()));
}

# 格式化参数并将输出写入范围
template <typename ArgFormatter, typename Char, typename Context>
typename Context::iterator vformat_to(
    typename ArgFormatter::iterator out, basic_string_view<Char> format_str,
    basic_format_args<Context> args,
    detail::locale_ref loc = detail::locale_ref()) {
  # 如果格式字符串的长度为2且内容为"{}"，则处理第一个参数
  if (format_str.size() == 2 && detail::equal2(format_str.data(), "{}")) {
    auto arg = args.get(0);
    if (!arg) detail::error_handler().on_error("argument not found");
    using iterator = typename ArgFormatter::iterator;
    return visit_format_arg(
        detail::default_arg_formatter<iterator, Char>{out, args, loc}, arg);
  }
  # 创建格式处理器并解析格式字符串
  detail::format_handler<ArgFormatter, Char, Context> h(out, format_str, args,
                                                        loc);
  detail::parse_format_string<false>(format_str, h);
  return h.context.out();
}

# 将指针转换为const void*，用于指针格式化
template <typename T> inline const void* ptr(const T* p) { return p; }
template <typename T> inline const void* ptr(const std::unique_ptr<T>& p) {
  return p.get();
}
template <typename T> inline const void* ptr(const std::shared_ptr<T>& p) {
  return p.get();
}

# 定义一个字节类，用于格式化
class bytes {
 private:
  string_view data_;
  friend struct formatter<bytes>;

 public:
  explicit bytes(string_view data) : data_(data) {}
};

# bytes类的格式化器
template <> struct formatter<bytes> {
 private:
  detail::dynamic_format_specs<char> specs_;

 public:
  # 解析格式规范
  template <typename ParseContext>
  FMT_CONSTEXPR auto parse(ParseContext& ctx) -> decltype(ctx.begin()) {
    using handler_type = detail::dynamic_specs_handler<ParseContext>;
  // 创建一个处理器对象，用于检查格式规范
  detail::specs_checker<handler_type> handler(handler_type(specs_, ctx),
                                              detail::type::string_type);
  // 解析格式规范，并返回迭代器
  auto it = parse_format_specs(ctx.begin(), ctx.end(), handler);
  // 检查字符串类型规范是否符合要求
  detail::check_string_type_spec(specs_.type, ctx.error_handler());
  // 返回迭代器
  return it;
}

template <typename FormatContext>
auto format(bytes b, FormatContext& ctx) -> decltype(ctx.out()) {
  // 处理动态规范中的宽度
  detail::handle_dynamic_spec<detail::width_checker>(specs_.width,
                                                     specs_.width_ref, ctx);
  // 处理动态规范中的精度
  detail::handle_dynamic_spec<detail::precision_checker>(
      specs_.precision, specs_.precision_ref, ctx);
  // 将字节数据写入输出流中，并返回结果
  return detail::write_bytes(ctx.out(), b.data_, specs_);
}
};
// 定义一个模板结构体 arg_join，继承自 detail::view
template <typename It, typename Sentinel, typename Char>
struct arg_join : detail::view {
  // 成员变量，表示迭代器范围的起始和结束，以及分隔符
  It begin;
  Sentinel end;
  basic_string_view<Char> sep;

  // 构造函数，接受迭代器范围的起始和结束，以及分隔符
  arg_join(It b, Sentinel e, basic_string_view<Char> s)
      : begin(b), end(e), sep(s) {}
};

// 定义一个 formatter 结构体，用于格式化 arg_join 类型的对象
template <typename It, typename Sentinel, typename Char>
struct formatter<arg_join<It, Sentinel, Char>, Char>
    : formatter<typename std::iterator_traits<It>::value_type, Char> {
  // format 方法，用于格式化 arg_join 类型的对象
  template <typename FormatContext>
  auto format(const arg_join<It, Sentinel, Char>& value, FormatContext& ctx)
      -> decltype(ctx.out()) {
    using base = formatter<typename std::iterator_traits<It>::value_type, Char>;
    auto it = value.begin;
    auto out = ctx.out();
    if (it != value.end) {
      out = base::format(*it++, ctx);
      while (it != value.end) {
        out = std::copy(value.sep.begin(), value.sep.end(), out);
        ctx.advance_to(out);
        out = base::format(*it++, ctx);
      }
    }
    return out;
  }
};

/**
  返回一个对象，用于将迭代器范围 `[begin, end)` 的元素以 `sep` 分隔格式化
 */
template <typename It, typename Sentinel>
arg_join<It, Sentinel, char> join(It begin, Sentinel end, string_view sep) {
  return {begin, end, sep};
}

// 重载的 join 函数，用于处理 wchar_t 类型的分隔符
template <typename It, typename Sentinel>
arg_join<It, Sentinel, wchar_t> join(It begin, Sentinel end, wstring_view sep) {
  return {begin, end, sep};
}

/**
  返回一个对象，用于将 range 的元素以 sep 分隔格式化

  **示例**::

    std::vector<int> v = {1, 2, 3};
    fmt::print("{}", fmt::join(v, ", "));
    // 输出: "1, 2, 3"

  ``fmt::join`` 可以应用传递的格式说明符到 range 的元素::

    fmt::print("{:02}", fmt::join(v, ", "));
    // 输出: "01, 02, 03"
 */
template <typename Range>
arg_join<detail::iterator_t<Range>, detail::sentinel_t<Range>, char> join(
    Range&& range, string_view sep) {
  return join(std::begin(range), std::end(range), sep);
}

template <typename Range>
// 定义一个名为 join 的函数模板，接受一个范围和一个分隔符，返回一个 arg_join 对象
arg_join<detail::iterator_t<Range>, detail::sentinel_t<Range>, wchar_t> join(
    Range&& range, wstring_view sep) {
  return join(std::begin(range), std::end(range), sep);
}

/**
  \rst
  将 *value* 使用类型 *T* 的默认格式转换为 ``std::string``。

  **示例**::

    #include <fmt/format.h>

    std::string answer = fmt::to_string(42);
  \endrst
 */
// 如果 T 不是整数类型，则定义一个 to_string 函数模板，接受一个值并返回一个 std::string
template <typename T, FMT_ENABLE_IF(!std::is_integral<T>::value)>
inline std::string to_string(const T& value) {
  std::string result;
  detail::write<char>(std::back_inserter(result), value);
  return result;
}

// 如果 T 是整数类型，则定义一个 to_string 函数模板，接受一个值并返回一个 std::string
template <typename T, FMT_ENABLE_IF(std::is_integral<T>::value)>
inline std::string to_string(T value) {
  // 缓冲区应足够大，以存储包括符号或布尔值“false”在内的数字
  constexpr int max_size = detail::digits10<T>() + 2;
  char buffer[max_size > 5 ? static_cast<unsigned>(max_size) : 5];
  char* begin = buffer;
  return std::string(begin, detail::write<char>(begin, value));
}

/**
  将 *value* 使用类型 *T* 的默认格式转换为 ``std::wstring``。
 */
// 定义一个 to_wstring 函数模板，接受一个值并返回一个 std::wstring
template <typename T> inline std::wstring to_wstring(const T& value) {
  return format(L"{}", value);
}

// 定义一个 to_string 函数模板，接受一个 basic_memory_buffer 对象并返回一个 std::basic_string
template <typename Char, size_t SIZE>
std::basic_string<Char> to_string(const basic_memory_buffer<Char, SIZE>& buf) {
  auto size = buf.size();
  detail::assume(size < std::basic_string<Char>().max_size());
  return std::basic_string<Char>(buf.data(), size);
}

// 定义一个 vformat_to 函数模板，接受一个 buffer 和 format_str，返回一个 buffer_appender 对象
template <typename Char>
detail::buffer_appender<Char> detail::vformat_to(
    detail::buffer<Char>& buf, basic_string_view<Char> format_str,
    basic_format_args<buffer_context<type_identity_t<Char>>> args) {
  using af = arg_formatter<typename buffer_context<Char>::iterator, Char>;
  return vformat_to<af>(buffer_appender<Char>(buf), format_str, args);
}

// 如果未定义 FMT_HEADER_ONLY，则声明一个 vformat_to 函数模板
#ifndef FMT_HEADER_ONLY
extern template format_context::iterator detail::vformat_to(
    detail::buffer<char>&, string_view, basic_format_args<format_context>);
namespace detail {
// 实例化模板函数，为 char 类型的 grouping_impl 函数声明
extern template FMT_API std::string grouping_impl<char>(locale_ref loc);
// 实例化模板函数，为 wchar_t 类型的 grouping_impl 函数声明
extern template FMT_API std::string grouping_impl<wchar_t>(locale_ref loc);
// 实例化模板函数，为 char 类型的 thousands_sep_impl 函数声明
extern template FMT_API char thousands_sep_impl<char>(locale_ref loc);
// 实例化模板函数，为 wchar_t 类型的 thousands_sep_impl 函数声明
extern template FMT_API wchar_t thousands_sep_impl<wchar_t>(locale_ref loc);
// 实例化模板函数，为 char 类型的 decimal_point_impl 函数声明
extern template FMT_API char decimal_point_impl(locale_ref loc);
// 实例化模板函数，为 wchar_t 类型的 decimal_point_impl 函数声明
extern template FMT_API wchar_t decimal_point_impl(locale_ref loc);
// 实例化模板函数，为 double 类型的 format_float 函数声明
extern template int format_float<double>(double value, int precision, float_specs specs, buffer<char>& buf);
// 实例化模板函数，为 long double 类型的 format_float 函数声明
extern template int format_float<long double>(long double value, int precision, float_specs specs, buffer<char>& buf);
// 删除 float 类型的 snprintf_float 函数声明
int snprintf_float(float value, int precision, float_specs specs, buffer<char>& buf) = delete;
// 实例化模板函数，为 double 类型的 snprintf_float 函数声明
extern template int snprintf_float<double>(double value, int precision, float_specs specs, buffer<char>& buf);
// 实例化模板函数，为 long double 类型的 snprintf_float 函数声明
extern template int snprintf_float<long double>(long double value, int precision, float_specs specs, buffer<char>& buf);
}  // namespace detail
#endif

// 根据输入的格式字符串和参数列表，将格式化后的字符串写入到缓冲区中
template <typename S, typename Char = char_t<S>, FMT_ENABLE_IF(detail::is_string<S>::value)>
inline typename FMT_BUFFER_CONTEXT(Char)::iterator vformat_to(
    detail::buffer<Char>& buf, const S& format_str,
    basic_format_args<FMT_BUFFER_CONTEXT(type_identity_t<Char>)> args) {
  return detail::vformat_to(buf, to_string_view(format_str), args);
}

// 根据输入的格式字符串和参数列表，将格式化后的字符串写入到缓冲区中
template <typename S, typename... Args, size_t SIZE = inline_buffer_size,
          typename Char = enable_if_t<detail::is_string<S>::value, char_t<S>>>
inline typename buffer_context<Char>::iterator format_to(
    # 格式化字符串和参数，生成格式化后的结果
      const auto& vargs = fmt::make_args_checked<Args...>(format_str, args...);
      # 调用 vformat_to 函数，将格式化后的结果写入到缓冲区中
      return detail::vformat_to(buf, to_string_view(format_str), vargs);
}
// 定义一个模板别名，用于输出格式化的上下文
template <typename OutputIt, typename Char = char>
using format_context_t = basic_format_context<OutputIt, Char>;

// 定义一个模板别名，用于格式化参数
template <typename OutputIt, typename Char = char>
using format_args_t = basic_format_args<format_context_t<OutputIt, Char>>;

// 定义一个模板别名，用于格式化输出到指定长度的上下文
template <typename OutputIt, typename Char = typename OutputIt::value_type>
using format_to_n_context FMT_DEPRECATED_ALIAS = buffer_context<Char>;

// 定义一个模板别名，用于格式化输出到指定长度的参数
template <typename OutputIt, typename Char = typename OutputIt::value_type>
using format_to_n_args FMT_DEPRECATED_ALIAS =
    basic_format_args<buffer_context<Char>>;

// 创建格式化输出到指定长度的参数
template <typename OutputIt, typename Char, typename... Args>
FMT_DEPRECATED format_arg_store<buffer_context<Char>, Args...>
make_format_to_n_args(const Args&... args) {
  return format_arg_store<buffer_context<Char>, Args...>(args...);
}

// 格式化输出到指定长度的字符串
template <typename Char, enable_if_t<(!std::is_same<Char, char>::value), int>>
std::basic_string<Char> detail::vformat(
    basic_string_view<Char> format_str,
    basic_format_args<buffer_context<type_identity_t<Char>>> args) {
  basic_memory_buffer<Char> buffer;
  detail::vformat_to(buffer, format_str, args);
  return to_string(buffer);
}

// 格式化输出到指定长度的宽字符流
template <typename Char, FMT_ENABLE_IF(std::is_same<Char, wchar_t>::value)>
void vprint(std::FILE* f, basic_string_view<Char> format_str,
            wformat_args args) {
  wmemory_buffer buffer;
  detail::vformat_to(buffer, format_str, args);
  buffer.push_back(L'\0');
  if (std::fputws(buffer.data(), f) == -1)
    FMT_THROW(system_error(errno, "cannot write to file"));
}

// 格式化输出到指定长度的宽字符流
template <typename Char, FMT_ENABLE_IF(std::is_same<Char, wchar_t>::value)>
void vprint(basic_string_view<Char> format_str, wformat_args args) {
  vprint(stdout, format_str, args);
}

// 如果使用了用户自定义字面量
#if FMT_USE_USER_DEFINED_LITERALS
namespace detail {

// 如果使用了UDL模板
#  if FMT_USE_UDL_TEMPLATE
// 定义一个UDL格式化器类
template <typename Char, Char... CHARS> class udl_formatter {
 public:
  // 重载()运算符，用于格式化输出
  template <typename... Args>
  std::basic_string<Char> operator()(Args&&... args) const {
    // 使用可变参数 CHARS 构建字符数组 s，并在末尾添加空字符
    static FMT_CONSTEXPR_DECL Char s[] = {CHARS..., '\0'};
    // 使用格式化字符串和参数 args 调用 format 函数，并返回结果
    return format(FMT_STRING(s), std::forward<Args>(args)...);
};
#  else
// 定义一个模板结构体 udl_formatter，用于处理用户定义的字面值字符串
template <typename Char> struct udl_formatter {
  basic_string_view<Char> str;

  // 重载 () 运算符，用于格式化字符串
  template <typename... Args>
  std::basic_string<Char> operator()(Args&&... args) const {
    return format(str, std::forward<Args>(args)...);
  }
};
#  endif  // FMT_USE_UDL_TEMPLATE

// 定义一个模板结构体 udl_arg，用于处理用户定义的字面值字符串
template <typename Char> struct udl_arg {
  const Char* str;

  // 重载 = 运算符，用于创建命名参数
  template <typename T> named_arg<Char, T> operator=(T&& value) const {
    return {str, std::forward<T>(value)};
  }
};
}  // namespace detail

// 内联命名空间 literals
inline namespace literals {
#  if FMT_USE_UDL_TEMPLATE
#    pragma GCC diagnostic push
#    pragma GCC diagnostic ignored "-Wpedantic"
#    if FMT_CLANG_VERSION
#      pragma GCC diagnostic ignored "-Wgnu-string-literal-operator-template"
#    endif
// 定义用户定义的字面值模板，用于处理用户定义的字面值字符串
template <typename Char, Char... CHARS>
FMT_CONSTEXPR detail::udl_formatter<Char, CHARS...> operator""_format() {
  return {};
}
#    pragma GCC diagnostic pop
#  else
/**
  \rst
  User-defined literal equivalent of :func:`fmt::format`.

  **Example**::

    using namespace fmt::literals;
    std::string message = "The answer is {}"_format(42);
  \endrst
 */
// 定义用户定义的字面值函数，用于处理用户定义的字面值字符串
FMT_CONSTEXPR detail::udl_formatter<char> operator"" _format(const char* s,
                                                             size_t n) {
  return {{s, n}};
}
FMT_CONSTEXPR detail::udl_formatter<wchar_t> operator"" _format(
    const wchar_t* s, size_t n) {
  return {{s, n}};
}
#  endif  // FMT_USE_UDL_TEMPLATE

/**
  \rst
  User-defined literal equivalent of :func:`fmt::arg`.

  **Example**::

    using namespace fmt::literals;
    fmt::print("Elapsed time: {s:.2f} seconds", "s"_a=1.23);
  \endrst
 */
// 定义用户定义的字面值函数，用于处理用户定义的字面值字符串
FMT_CONSTEXPR detail::udl_arg<char> operator"" _a(const char* s, size_t) {
  return {s};
}
FMT_CONSTEXPR detail::udl_arg<wchar_t> operator"" _a(const wchar_t* s, size_t) {
  return {s};
}
}  // namespace literals
#endif  // FMT_USE_USER_DEFINED_LITERALS
FMT_END_NAMESPACE

#ifdef FMT_HEADER_ONLY
#  define FMT_FUNC inline
#  include "format-inl.h"
#else
#  define FMT_FUNC
#endif
#endif  // FMT_FORMAT_H_
```