# `xmrig\src\3rdparty\epee\span.h`

```cpp
// 版权声明
// 版权声明，允许在源代码和二进制形式中重新分发和使用，但需要满足以下条件
// 1. 源代码的重新分发必须保留上述版权声明、条件列表和以下免责声明
// 2. 二进制形式的重新分发必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明
// 3. 未经特定事先书面许可，不得使用版权持有者或其贡献者的名称来认可或推广从本软件派生的产品
// 版权持有者和贡献者提供的本软件是"按原样"提供的，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的暗示担保。在任何情况下，无论是合同、严格责任还是侵权（包括疏忽或其他方式）产生的任何理论责任，版权持有者或贡献者均不对任何直接、间接、附带、特殊、示范性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）负责，即使已被告知可能发生此类损害。

// 防止头文件被多次包含
#pragma once

// 引入标准库头文件
#include <algorithm>
#include <cstdint>
#include <memory>
#include <string>
#include <type_traits>

// 命名空间定义
namespace epee
{
  /*!
    \brief 非拥有数据序列。不进行深拷贝

    受 `gsl::span` 和/或 `boost::iterator_range` 启发。此类旨在用作需要接受非拥有数据序列的函数的参数类型
  // 定义一个模板类 span，表示可写或只读的数据序列，最常见的情况是 span<char> 和 span<std::uint8_t>。如果明确文档说明不进行深拷贝，才推荐将其用作类成员。C 数组很容易转换为这种类型。

  // 转换自 C 字符串字面量到 span<const char> 会包括空终止符。
  // 从派生类型到基类指针的转换是不允许的；派生类型的数组不是基类类型的数组。
  template<typename T>
  class span
  {
    template<typename U>
    static constexpr bool safe_conversion() noexcept
    {
      // 允许精确匹配或者 T* -> const T*。
      using with_const = typename std::add_const<U>::type;
      return std::is_same<T, U>() ||
        (std::is_const<T>() && std::is_same<T, with_const>());
    }

  public:
    using value_type = T;
    using size_type = std::size_t;
    using difference_type = std::ptrdiff_t;
    using pointer = T*;
    using const_pointer = const T*;
    using reference = T&;
    using const_reference = const T&;
    using iterator = pointer;
    using const_iterator = const_pointer;

    // 默认构造函数，初始化为空指针和长度为 0
    constexpr span() noexcept : ptr(nullptr), len(0) {}
    // 接受空指针的构造函数，调用默认构造函数
    constexpr span(std::nullptr_t) noexcept : span() {}

    //! 防止派生到基类的转换；在这个上下文中是无效的。
    template<typename U, typename = typename std::enable_if<safe_conversion<U>()>::type>
    // 接受指针和长度的构造函数
    constexpr span(U* const src_ptr, const std::size_t count) noexcept
      : ptr(src_ptr), len(count) {}

    //! 从 C 数组转换。防止使用 sizeof + 数组导致的常见错误。
    template<std::size_t N>
    // 接受数组引用的构造函数
    constexpr span(T (&src)[N]) noexcept : span(src, N) {}

    // 拷贝构造函数，默认实现
    constexpr span(const span&) noexcept = default;
    // 拷贝赋值运算符，默认实现
    span& operator=(const span&) noexcept = default;

    /*! 尝试从 span 的开头移除 amount 个元素。
    \return 移除的元素数量。 */
    // 从 span 的开头移除指定数量的元素
    std::size_t remove_prefix(std::size_t amount) noexcept
    {
        // 确保要复制的长度不超过实际长度
        amount = std::min(len, amount);
        // 移动指针到下一个位置
        ptr += amount;
        // 更新剩余长度
        len -= amount;
        // 返回已经复制的长度
        return amount;
    }

    // 返回指向数据的迭代器
    constexpr iterator begin() const noexcept { return ptr; }
    // 返回指向数据的常量迭代器
    constexpr const_iterator cbegin() const noexcept { return ptr; }

    // 返回指向数据末尾的迭代器
    constexpr iterator end() const noexcept { return begin() + size(); }
    // 返回指向数据末尾的常量迭代器
    constexpr const_iterator cend() const noexcept { return cbegin() + size(); }

    // 检查数据是否为空
    constexpr bool empty() const noexcept { return size() == 0; }
    // 返回指向数据的指针
    constexpr pointer data() const noexcept { return ptr; }
    // 返回数据的大小
    constexpr std::size_t size() const noexcept { return len; }
    // 返回数据的字节大小
    constexpr std::size_t size_bytes() const noexcept { return size() * sizeof(value_type); }

    // 重载操作符，返回指定索引位置的数据的引用
    T &operator[](size_t idx) noexcept { return ptr[idx]; }
    // 重载操作符，返回指定索引位置的数据的常量引用
    const T &operator[](size_t idx) const noexcept { return ptr[idx]; }

  private:
    T* ptr; // 指向数据的指针
    std::size_t len; // 数据的长度
  };

  //! \return `span<const T::value_type>` from a STL compatible `src`.
  // 将 STL 兼容的容器转换为只读的 span
  template<typename T>
  constexpr span<const typename T::value_type> to_span(const T& src)
  {
    // 编译器会在 size() 不是 size_t 类型时提供诊断信息
    return {src.data(), src.size()};
  }

  //! \return `span<T::value_type>` from a STL compatible `src`.
  // 将 STL 兼容的容器转换为可变的 span
  template<typename T>
  constexpr span<typename T::value_type> to_mut_span(T& src)
  {
    // 编译器会在 size() 不是 size_t 类型时提供诊断信息
    return {src.data(), src.size()};
  }

  // 检查类型 T 是否有填充
  template<typename T>
  constexpr bool has_padding() noexcept
  {
    // 如果类型 T 不是标准布局或者对齐不是 1，返回 true
    return !std::is_standard_layout<T>() || alignof(T) != 1;
  }

  //! \return Cast data from `src` as `span<const std::uint8_t>`.
  // 将数据从 src 转换为只读的 std::uint8_t 类型的 span
  template<typename T>
  span<const std::uint8_t> to_byte_span(const span<const T> src) noexcept
  {
    // 静态断言，如果源类型可能有填充，会提供错误信息
    static_assert(!has_padding<T>(), "source type may have padding");
  // 返回一个指向src数据的const std::uint8_t指针和数据的大小
  return {reinterpret_cast<const std::uint8_t*>(src.data()), src.size_bytes()};
}

  // 返回一个span<const std::uint8_t>，表示&src处的字节
  template<typename T>
  span<const std::uint8_t> as_byte_span(const T& src) noexcept
  {
    static_assert(!std::is_empty<T>(), "empty types will not work -> sizeof == 1");
    static_assert(!has_padding<T>(), "source type may have padding");
    return {reinterpret_cast<const std::uint8_t*>(std::addressof(src)), sizeof(T)};
  }

  // 返回一个span<std::uint8_t>，表示&src处的字节
  template<typename T>
  span<std::uint8_t> as_mut_byte_span(T& src) noexcept
  {
    static_assert(!std::is_empty<T>(), "empty types will not work -> sizeof == 1");
    static_assert(!has_padding<T>(), "source type may have padding");
    return {reinterpret_cast<std::uint8_t*>(std::addressof(src)), sizeof(T)};
  }

  // 从std::string创建一个span
  template<typename T>
  span<const T> strspan(const std::string &s) noexcept
  {
    static_assert(std::is_same<T, char>() || std::is_same<T, unsigned char>() || std::is_same<T, int8_t>() || std::is_same<T, uint8_t>(), "Unexpected type");
    return {reinterpret_cast<const T*>(s.data()), s.size()};
  }
# 闭合前面的函数定义
```