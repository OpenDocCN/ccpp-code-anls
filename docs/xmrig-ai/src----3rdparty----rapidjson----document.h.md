# `xmrig\src\3rdparty\rapidjson\document.h`

```cpp
// 定义了一个宏，用于防止重复包含头文件
#ifndef RAPIDJSON_DOCUMENT_H_
#define RAPIDJSON_DOCUMENT_H_

// 包含了文件 reader.h，meta.h，strfunc.h，memorystream.h，encodedstream.h，以及一些标准库头文件
#include "reader.h"
#include "internal/meta.h"
#include "internal/strfunc.h"
#include "memorystream.h"
#include "encodedstream.h"
#include <new>      // placement new
#include <limits>
#ifdef __cpp_lib_three_way_comparison
#include <compare>
#endif

// 关闭一些编译器警告
RAPIDJSON_DIAG_PUSH
#ifdef __clang__
RAPIDJSON_DIAG_OFF(padded)
RAPIDJSON_DIAG_OFF(switch-enum)
RAPIDJSON_DIAG_OFF(c++98-compat)
#elif defined(_MSC_VER)
RAPIDJSON_DIAG_OFF(4127) // conditional expression is constant
RAPIDJSON_DIAG_OFF(4244) // conversion from kXxxFlags to 'uint16_t', possible loss of data
#endif

#ifdef __GNUC__
RAPIDJSON_DIAG_OFF(effc++)
#endif // __GNUC__

#ifdef GetObject
// 处理一个已知问题，避免宏 GetObject 被重新定义
#pragma push_macro("GetObject")
#define RAPIDJSON_WINDOWS_GETOBJECT_WORKAROUND_APPLIED
#undef GetObject
#endif

// 如果未定义 RAPIDJSON_NOMEMBERITERATORCLASS，则包含头文件 iterator
#ifndef RAPIDJSON_NOMEMBERITERATORCLASS
#include <iterator> // std::random_access_iterator_tag
#endif

// 如果定义了 RAPIDJSON_USE_MEMBERSMAP，则包含头文件 map
#if RAPIDJSON_USE_MEMBERSMAP
#include <map> // std::multimap
#endif

// 命名空间的开始
RAPIDJSON_NAMESPACE_BEGIN

// 前向声明
// 定义一个模板类 GenericValue，具有编码和分配器类型参数
template <typename Encoding, typename Allocator>
class GenericValue;

// 定义一个模板类 GenericDocument，具有编码、分配器和堆栈分配器类型参数
template <typename Encoding, typename Allocator, typename StackAllocator>
class GenericDocument;

// 如果未定义 RAPIDJSON_DEFAULT_ALLOCATOR，则使用 MemoryPoolAllocator<CrtAllocator> 作为默认分配器
#ifndef RAPIDJSON_DEFAULT_ALLOCATOR
#define RAPIDJSON_DEFAULT_ALLOCATOR MemoryPoolAllocator<CrtAllocator>
#endif

// 如果未定义 RAPIDJSON_DEFAULT_STACK_ALLOCATOR，则使用 CrtAllocator 作为默认堆栈分配器
#ifndef RAPIDJSON_DEFAULT_STACK_ALLOCATOR
#define RAPIDJSON_DEFAULT_STACK_ALLOCATOR CrtAllocator
#endif

// 如果未定义 RAPIDJSON_VALUE_DEFAULT_OBJECT_CAPACITY，则默认对象容量为 16
#ifndef RAPIDJSON_VALUE_DEFAULT_OBJECT_CAPACITY
// 默认 rapidjson::Value 分配内存的对象数量
#define RAPIDJSON_VALUE_DEFAULT_OBJECT_CAPACITY 16
#endif

// 如果未定义 RAPIDJSON_VALUE_DEFAULT_ARRAY_CAPACITY，则默认数组容量为 16
#ifndef RAPIDJSON_VALUE_DEFAULT_ARRAY_CAPACITY
// 默认 rapidjson::Value 分配内存的数组元素数量
#define RAPIDJSON_VALUE_DEFAULT_ARRAY_CAPACITY 16
#endif

// JSON 对象值中的名称-值对
template <typename Encoding, typename Allocator>
class GenericMember {
public:
    GenericValue<Encoding, Allocator> name;     //!< 成员的名称（必须是字符串）
    // 创建一个通用类型的值，使用指定的编码和分配器
    GenericValue<Encoding, Allocator> value;    //!< value of member.
#if RAPIDJSON_HAS_CXX11_RVALUE_REFS
    //! 如果支持 C++11 的右值引用，则定义移动构造函数
    GenericMember(GenericMember&& rhs) RAPIDJSON_NOEXCEPT
        : name(std::move(rhs.name)),  // 使用移动语义获取名称
          value(std::move(rhs.value))  // 使用移动语义获取值
    {
    }

    //! 如果支持 C++11 的右值引用，则定义移动赋值运算符
    GenericMember& operator=(GenericMember&& rhs) RAPIDJSON_NOEXCEPT {
        return *this = static_cast<GenericMember&>(rhs);
    }
#endif

    //! 使用移动语义进行赋值操作
    /*! \param rhs 赋值的来源。赋值后，其名称和值将变为 null 值。
    */
    GenericMember& operator=(GenericMember& rhs) RAPIDJSON_NOEXCEPT {
        if (RAPIDJSON_LIKELY(this != &rhs)) {
            name = rhs.name;  // 赋值名称
            value = rhs.value;  // 赋值值
        }
        return *this;
    }

    // 为 std::sort() 和其他潜在的 STL 使用定义 swap() 函数
    friend inline void swap(GenericMember& a, GenericMember& b) RAPIDJSON_NOEXCEPT {
        a.name.Swap(b.name);  // 交换名称
        a.value.Swap(b.value);  // 交换值
    }

private:
    //! 禁止复制构造函数
    GenericMember(const GenericMember& rhs);
};

///////////////////////////////////////////////////////////////////////////////
// GenericMemberIterator

#ifndef RAPIDJSON_NOMEMBERITERATORCLASS

//! JSON 对象值的（常量）成员迭代器
/*!
    \tparam Const 是否为常量迭代器？
    \tparam Encoding    值的编码。（即使非字符串值在文档中也需要具有相同的编码）
    \tparam Allocator   用于分配对象、数组和字符串内存的分配器类型。

    该类实现了 GenericValue 的 GenericMember 元素的随机访问迭代器，
    参见 ISO/IEC 14882:2003(E) C++ 标准，24.1 [lib.iterator.requirements]。

    \note 此迭代器实现主要用于避免从迭代器值到 \c NULL 的隐式转换，
        例如从 GenericValue::FindMember。
    # 如果你的平台不提供 C++ <iterator> 头文件，可以定义 RAPIDJSON_NOMEMBERITERATORCLASS 为指针实现来回退
    # 参见 GenericMember, GenericValue::MemberIterator, GenericValue::ConstMemberIterator
template <bool Const, typename Encoding, typename Allocator>
class GenericMemberIterator {

    friend class GenericValue<Encoding,Allocator>;
    template <bool, typename, typename> friend class GenericMemberIterator;

    typedef GenericMember<Encoding,Allocator> PlainType;
    typedef typename internal::MaybeAddConst<Const,PlainType>::Type ValueType;

public:
    //! Iterator type itself
    typedef GenericMemberIterator Iterator;
    //! Constant iterator type
    typedef GenericMemberIterator<true,Encoding,Allocator>  ConstIterator;
    //! Non-constant iterator type
    typedef GenericMemberIterator<false,Encoding,Allocator> NonConstIterator;

    /** \name std::iterator_traits support */
    //@{
    typedef ValueType      value_type;  // 定义 value_type 为 ValueType
    typedef ValueType *    pointer;     // 定义 pointer 为指向 ValueType 的指针
    typedef ValueType &    reference;   // 定义 reference 为 ValueType 的引用
    typedef std::ptrdiff_t difference_type;  // 定义 difference_type 为 std::ptrdiff_t
    typedef std::random_access_iterator_tag iterator_category;  // 定义 iterator_category 为 std::random_access_iterator_tag
    //@}

    //! Pointer to (const) GenericMember
    typedef pointer         Pointer;  // 定义 Pointer 为指针类型
    //! Reference to (const) GenericMember
    typedef reference       Reference;  // 定义 Reference 为引用类型
    //! Signed integer type (e.g. \c ptrdiff_t)
    typedef difference_type DifferenceType;  // 定义 DifferenceType 为 difference_type 类型

    //! Default constructor (singular value)
    /*! Creates an iterator pointing to no element.
        \note All operations, except for comparisons, are undefined on such values.
     */
    GenericMemberIterator() : ptr_() {}  // 默认构造函数，创建一个指向空元素的迭代器

    //! Iterator conversions to more const
    /*!
        \param it (Non-const) iterator to copy from

        Allows the creation of an iterator from another GenericMemberIterator
        that is "less const".  Especially, creating a non-constant iterator
        from a constant iterator are disabled:
        \li const -> non-const (not ok)
        \li const -> const (ok)
        \li non-const -> const (ok)
        \li non-const -> non-const (ok)

        \note If the \c Const template parameter is already \c false, this
            constructor effectively defines a regular copy-constructor.
            Otherwise, the copy constructor is implicitly defined.
    */
    // 从另一个“更不稳定”的 GenericMemberIterator 创建一个迭代器
    // 特别是，禁止从常量迭代器创建非常量迭代器：
    // const -> non-const (不允许)
    // const -> const (允许)
    // non-const -> const (允许)
    // non-const -> non-const (允许)
    GenericMemberIterator(const NonConstIterator & it) : ptr_(it.ptr_) {}
    // 重载赋值运算符，将非常量迭代器赋值给当前迭代器
    Iterator& operator=(const NonConstIterator & it) { ptr_ = it.ptr_; return *this; }

    //! @name stepping
    //@{
    // 重载前置递增运算符
    Iterator& operator++(){ ++ptr_; return *this; }
    // 重载前置递减运算符
    Iterator& operator--(){ --ptr_; return *this; }
    // 重载后置递增运算符
    Iterator  operator++(int){ Iterator old(*this); ++ptr_; return old; }
    // 重载后置递减运算符
    Iterator  operator--(int){ Iterator old(*this); --ptr_; return old; }
    //@}

    //! @name increment/decrement
    //@{
    // 重载加法运算符
    Iterator operator+(DifferenceType n) const { return Iterator(ptr_+n); }
    // 重载减法运算符
    Iterator operator-(DifferenceType n) const { return Iterator(ptr_-n); }

    // 重载复合赋值加法运算符
    Iterator& operator+=(DifferenceType n) { ptr_+=n; return *this; }
    // 重载复合赋值减法运算符
    Iterator& operator-=(DifferenceType n) { ptr_-=n; return *this; }
    //@}

    //! @name relations
    //@{
    // 模板函数，重载相等运算符
    template <bool Const_> bool operator==(const GenericMemberIterator<Const_, Encoding, Allocator>& that) const { return ptr_ == that.ptr_; }
    // 模板函数，重载不等运算符
    template <bool Const_> bool operator!=(const GenericMemberIterator<Const_, Encoding, Allocator>& that) const { return ptr_ != that.ptr_; }
    // 模板函数，重载小于等于运算符
    template <bool Const_> bool operator<=(const GenericMemberIterator<Const_, Encoding, Allocator>& that) const { return ptr_ <= that.ptr_; }
    # 定义模板函数，用于比较当前迭代器指向的元素是否大于等于给定迭代器指向的元素
    template <bool Const_> bool operator>=(const GenericMemberIterator<Const_, Encoding, Allocator>& that) const { return ptr_ >= that.ptr_; }
    # 定义模板函数，用于比较当前迭代器指向的元素是否小于给定迭代器指向的元素
    template <bool Const_> bool operator< (const GenericMemberIterator<Const_, Encoding, Allocator>& that) const { return ptr_ < that.ptr_; }
    # 定义模板函数，用于比较当前迭代器指向的元素是否大于给定迭代器指向的元素
    template <bool Const_> bool operator> (const GenericMemberIterator<Const_, Encoding, Allocator>& that) const { return ptr_ > that.ptr_; }
#ifdef __cpp_lib_three_way_comparison
    // 如果支持三路比较运算符，则重载比较运算符
    template <bool Const_> std::strong_ordering operator<=>(const GenericMemberIterator<Const_, Encoding, Allocator>& that) const { return ptr_ <=> that.ptr_; }
#endif
    //@}

    //! @name dereference
    //@{
    // 解引用操作符重载，返回引用
    Reference operator*() const { return *ptr_; }
    // 指向成员访问操作符重载，返回指针
    Pointer   operator->() const { return ptr_; }
    // 下标操作符重载，返回引用
    Reference operator[](DifferenceType n) const { return ptr_[n]; }
    //@}

    //! Distance
    // 重载减法操作符，返回两个迭代器之间的距离
    DifferenceType operator-(ConstIterator that) const { return ptr_-that.ptr_; }

private:
    //! Internal constructor from plain pointer
    // 显式的内部构造函数，用于从普通指针创建对象
    explicit GenericMemberIterator(Pointer p) : ptr_(p) {}

    Pointer ptr_; //!< raw pointer // 原始指针
};

#else // RAPIDJSON_NOMEMBERITERATORCLASS

// class-based member iterator implementation disabled, use plain pointers

template <bool Const, typename Encoding, typename Allocator>
class GenericMemberIterator;

//! non-const GenericMemberIterator
template <typename Encoding, typename Allocator>
class GenericMemberIterator<false,Encoding,Allocator> {
public:
    //! use plain pointer as iterator type
    // 使用普通指针作为迭代器类型
    typedef GenericMember<Encoding,Allocator>* Iterator;
};
//! const GenericMemberIterator
template <typename Encoding, typename Allocator>
class GenericMemberIterator<true,Encoding,Allocator> {
public:
    //! use plain const pointer as iterator type
    // 使用普通 const 指针作为迭代器类型
    typedef const GenericMember<Encoding,Allocator>* Iterator;
};

#endif // RAPIDJSON_NOMEMBERITERATORCLASS

///////////////////////////////////////////////////////////////////////////////
// GenericStringRef

//! Reference to a constant string (not taking a copy)
/*!
    \tparam CharType character type of the string

    This helper class is used to automatically infer constant string
    references for string literals, especially from \c const \b (!)
    character arrays.

    The main use is for creating JSON string values without copying the
    source string via an \ref Allocator.  This requires that the referenced
    # 确保字符串指针具有足够的生命周期，超过关联的GenericValue的生命周期。
    
    # 示例
    # 创建一个值为"foo"的Value对象，不需要复制和计算长度
    Value v("foo");   // ok, no need to copy & calculate length
    # 创建一个名为foo的常量字符数组
    const char foo[] = "foo";
    # 将foo的值设置为v的字符串，可以
    v.SetString(foo); // ok

    # 创建一个指向foo的指针bar
    const char* bar = foo;
    # 创建一个Value对象x，使用bar的生命周期不可靠，不可以
    // Value x(bar); // not ok, can't rely on bar's lifetime
    # 创建一个Value对象x，使用StringRef显式保证生命周期
    Value x(StringRef(bar)); // lifetime explicitly guaranteed by user
    # 创建一个Value对象y，使用StringRef显式传递长度
    Value y(StringRef(bar, 3));  // ok, explicitly pass length
    
    # 参见StringRef, GenericValue::SetString
    \see StringRef, GenericValue::SetString
/*
    为通用字符串引用定义结构模板
*/
template<typename CharType>
struct GenericStringRef {
    typedef CharType Ch; //!< 字符串的字符类型

    //! 从 \c const 字符数组创建字符串引用
#ifndef __clang__ // -Wdocumentation
    /*!
        此构造函数隐式地从 \c const 字符数组创建一个常量字符串引用。
        它比 \ref StringRef(const CharType*) 具有更好的性能，通过从数组长度推断字符串 \ref length，
        并且还支持包含空字符的字符串。

        \tparam N 字符串的长度，自动推断

        \param str 常量字符数组，生命周期假定比字符串在例如 GenericValue 中的使用更长

        \post \ref s == str

        \note 常量复杂度。
        \note 有一个隐藏的、私有的重载，禁止通过此构造函数创建对非 const 字符数组的引用。
            因此，例如通过 \c snprintf 填充的函数作用域数组被排除在考虑范围之外。
            在这种情况下，引用的字符串应该被 \b 复制到 GenericValue 中。
     */
#endif
    template<SizeType N>
    GenericStringRef(const CharType (&str)[N]) RAPIDJSON_NOEXCEPT
        : s(str), length(N-1) {}

    //! 显式地从 \c const 字符指针创建字符串引用
#ifndef __clang__ // -Wdocumentation
    /*!
        这个构造函数可以被用来显式地创建一个指向常量字符串指针的引用。

        \see StringRef(const CharType*)

        \param str 常量字符指针，假定其生命周期比字符串在 GenericValue 中的使用更长

        \post \ref s == str

        \note 有一个隐藏的、私有的重载，禁止通过这个构造函数创建对非常量字符数组的引用。
            因此，例如通过 \c snprintf 填充的函数作用域数组被排除在考虑范围之外。
            在这种情况下，被引用的字符串应该被 \b 复制到 GenericValue 中。
     */
#endif
    // 从指针创建常量字符串引用
    explicit GenericStringRef(const CharType* str)
        : s(str), length(NotNullStrLen(str)) {}

    //! 从指针和长度创建常量字符串引用
#ifndef __clang__ // -Wdocumentation
    /*! \param str 常量字符串，假定其生命周期长于在 JSON GenericValue 对象中使用字符串的生命周期
        \param len 字符串的长度，不包括结尾的 NULL 终止符

        \post \ref s == str && \ref length == len
        \note 常量复杂度。
     */
#endif
    // 从指针和长度创建常量字符串引用
    GenericStringRef(const CharType* str, SizeType len)
        : s(RAPIDJSON_LIKELY(str) ? str : emptyString), length(len) { RAPIDJSON_ASSERT(str != 0 || len == 0u); }

    // 从另一个 GenericStringRef 对象创建常量字符串引用
    GenericStringRef(const GenericStringRef& rhs) : s(rhs.s), length(rhs.length) {}

    //! 隐式转换为普通的 CharType 指针
    operator const Ch *() const { return s; }

    const Ch* const s; //!< 普通的 CharType 指针
    const SizeType length; //!< 字符串的长度（不包括结尾的 NULL 终止符）

private:
    // 返回非空字符串的长度
    SizeType NotNullStrLen(const CharType* str) {
        RAPIDJSON_ASSERT(str != 0);
        return internal::StrLen(str);
    }

    /// 空字符串 - 当传入 NULL 指针时使用
    static const Ch emptyString[];

    //! 禁止从非常量数组构造
    template<SizeType N>
    GenericStringRef(CharType (&str)[N]) /* = delete */;
    //! 不允许复制赋值操作符 - 不可变类型
    GenericStringRef& operator=(const GenericStringRef& rhs) /* = delete */;
};

template<typename CharType>
const CharType GenericStringRef<CharType>::emptyString[] = { CharType() };

//! 将字符指针标记为常量字符串
/*! 将普通字符指针标记为“字符串字面值”。如果字符串的生命周期足够长，可以使用此函数
    避免将字符字符串复制为 JSON GenericValue 对象中的值的引用。
    \tparam CharType 字符串的字符类型
    # 参数说明：常量字符串，假设其生命周期长于在 GenericValue 中使用的字符串
    # 返回值：GenericStringRef 字符串引用对象
    # 关联：GenericStringRef
    # 参见：GenericValue::GenericValue(StringRefType), GenericValue::operator=(StringRefType), GenericValue::SetString(StringRefType), GenericValue::PushBack(StringRefType, Allocator&), GenericValue::AddMember
template<typename CharType>
inline GenericStringRef<CharType> StringRef(const CharType* str) {
    return GenericStringRef<CharType>(str);
}

// 定义了一个模板函数，用于将字符指针标记为常量字符串，并返回一个字符串引用对象


template<typename CharType>
inline GenericStringRef<CharType> StringRef(const CharType* str, size_t length) {
    return GenericStringRef<CharType>(str, SizeType(length));
}

// 定义了一个模板函数，用于将字符指针标记为常量字符串，并返回一个字符串引用对象，支持指定长度的字符串


#if RAPIDJSON_HAS_STDSTRING
template<typename CharType>
inline GenericStringRef<CharType> StringRef(const std::basic_string<CharType>& str) {
    return GenericStringRef<CharType>(str.data(), SizeType(str.size()));
}
#endif

// 如果定义了宏 RAPIDJSON_HAS_STDSTRING，则定义了一个模板函数，用于将字符串对象标记为常量字符串，并返回一个字符串引用对象


///////////////////////////////////////////////////////////////////////////////

// 分隔符，表示以上代码块结束
// 定义了用于通用值类型的特征
namespace internal {

// 用于判断是否为通用值类型的模板结构体
template <typename T, typename Encoding = void, typename Allocator = void>
struct IsGenericValueImpl : FalseType {};

// 根据嵌套的编码和分配器类型选择候选项
template <typename T> struct IsGenericValueImpl<T, typename Void<typename T::EncodingType>::Type, typename Void<typename T::AllocatorType>::Type>
    : IsBaseOf<GenericValue<typename T::EncodingType, typename T::AllocatorType>, T>::Type {};

// 用于匹配任意通用值实例的辅助结构体，包括派生类
template <typename T> struct IsGenericValue : IsGenericValueImpl<T>::Type {};

} // namespace internal

///////////////////////////////////////////////////////////////////////////////
// TypeHelper

namespace internal {

// 用于帮助处理不同类型的值的模板结构体
template <typename ValueType, typename T>
struct TypeHelper {};

// 用于处理布尔类型值的模板结构体
template<typename ValueType>
struct TypeHelper<ValueType, bool> {
    static bool Is(const ValueType& v) { return v.IsBool(); }
    static bool Get(const ValueType& v) { return v.GetBool(); }
    static ValueType& Set(ValueType& v, bool data) { return v.SetBool(data); }
    static ValueType& Set(ValueType& v, bool data, typename ValueType::AllocatorType&) { return v.SetBool(data); }
};

// 用于处理整数类型值的模板结构体
template<typename ValueType>
struct TypeHelper<ValueType, int> {
    static bool Is(const ValueType& v) { return v.IsInt(); }
    static int Get(const ValueType& v) { return v.GetInt(); }
    static ValueType& Set(ValueType& v, int data) { return v.SetInt(data); }
    static ValueType& Set(ValueType& v, int data, typename ValueType::AllocatorType&) { return v.SetInt(data); }
};

// 用于处理无符号整数类型值的模板结构体
template<typename ValueType>
struct TypeHelper<ValueType, unsigned> {
    static bool Is(const ValueType& v) { return v.IsUint(); }
    static unsigned Get(const ValueType& v) { return v.GetUint(); }
    static ValueType& Set(ValueType& v, unsigned data) { return v.SetUint(data); }
    // 静态函数，用于设置值类型对象的值为无符号整数，并返回设置后的对象
    static ValueType& Set(ValueType& v, unsigned data, typename ValueType::AllocatorType&) { return v.SetUint(data); }
};

#ifdef _MSC_VER
// 确保 long 类型的大小和 int 类型相同
RAPIDJSON_STATIC_ASSERT(sizeof(long) == sizeof(int));
// 定义 long 类型的辅助结构体
template<typename ValueType>
struct TypeHelper<ValueType, long> {
    // 判断值是否为 int 类型
    static bool Is(const ValueType& v) { return v.IsInt(); }
    // 获取值并转换为 long 类型
    static long Get(const ValueType& v) { return v.GetInt(); }
    // 设置值为 long 类型
    static ValueType& Set(ValueType& v, long data) { return v.SetInt(data); }
    // 设置值为 long 类型，带分配器
    static ValueType& Set(ValueType& v, long data, typename ValueType::AllocatorType&) { return v.SetInt(data); }
};

// 确保 unsigned long 类型的大小和 unsigned 类型相同
RAPIDJSON_STATIC_ASSERT(sizeof(unsigned long) == sizeof(unsigned));
// 定义 unsigned long 类型的辅助结构体
template<typename ValueType>
struct TypeHelper<ValueType, unsigned long> {
    // 判断值是否为 unsigned int 类型
    static bool Is(const ValueType& v) { return v.IsUint(); }
    // 获取值并转换为 unsigned long 类型
    static unsigned long Get(const ValueType& v) { return v.GetUint(); }
    // 设置值为 unsigned long 类型
    static ValueType& Set(ValueType& v, unsigned long data) { return v.SetUint(data); }
    // 设置值为 unsigned long 类型，带分配器
    static ValueType& Set(ValueType& v, unsigned long data, typename ValueType::AllocatorType&) { return v.SetUint(data); }
};
#endif

// 定义 int64_t 类型的辅助结构体
template<typename ValueType>
struct TypeHelper<ValueType, int64_t> {
    // 判断值是否为 int64 类型
    static bool Is(const ValueType& v) { return v.IsInt64(); }
    // 获取值并转换为 int64_t 类型
    static int64_t Get(const ValueType& v) { return v.GetInt64(); }
    // 设置值为 int64_t 类型
    static ValueType& Set(ValueType& v, int64_t data) { return v.SetInt64(data); }
    // 设置值为 int64_t 类型，带分配器
    static ValueType& Set(ValueType& v, int64_t data, typename ValueType::AllocatorType&) { return v.SetInt64(data); }
};

// 定义 uint64_t 类型的辅助结构体
template<typename ValueType>
struct TypeHelper<ValueType, uint64_t> {
    // 判断值是否为 uint64 类型
    static bool Is(const ValueType& v) { return v.IsUint64(); }
    // 获取值并转换为 uint64_t 类型
    static uint64_t Get(const ValueType& v) { return v.GetUint64(); }
    // 设置值为 uint64_t 类型
    static ValueType& Set(ValueType& v, uint64_t data) { return v.SetUint64(data); }
    // 设置值为 uint64_t 类型，带分配器
    static ValueType& Set(ValueType& v, uint64_t data, typename ValueType::AllocatorType&) { return v.SetUint64(data); }
};

// 定义 double 类型的辅助结构体
template<typename ValueType>
struct TypeHelper<ValueType, double> {
    // 判断值是否为 double 类型
    static bool Is(const ValueType& v) { return v.IsDouble(); }
    // 获取值并转换为 double 类型
    static double Get(const ValueType& v) { return v.GetDouble(); }
    # 设置值为 double 类型的数据，并返回该值
    static ValueType& Set(ValueType& v, double data) { return v.SetDouble(data); }
    # 设置值为 double 类型的数据，并返回该值，同时传入自定义的分配器
    static ValueType& Set(ValueType& v, double data, typename ValueType::AllocatorType&) { return v.SetDouble(data); }
// 结构模板 TypeHelper 的特化，用于处理 float 类型的值
template<typename ValueType>
struct TypeHelper<ValueType, float> {
    // 检查值是否为 float 类型
    static bool Is(const ValueType& v) { return v.IsFloat(); }
    // 获取值的 float 数据
    static float Get(const ValueType& v) { return v.GetFloat(); }
    // 设置值为 float 数据
    static ValueType& Set(ValueType& v, float data) { return v.SetFloat(data); }
    // 设置值为 float 数据，使用指定的分配器
    static ValueType& Set(ValueType& v, float data, typename ValueType::AllocatorType&) { return v.SetFloat(data); }
};

// 结构模板 TypeHelper 的特化，用于处理 const char* 类型的值
template<typename ValueType>
struct TypeHelper<ValueType, const typename ValueType::Ch*> {
    // 定义 const char* 类型的别名
    typedef const typename ValueType::Ch* StringType;
    // 检查值是否为字符串类型
    static bool Is(const ValueType& v) { return v.IsString(); }
    // 获取值的 const char* 数据
    static StringType Get(const ValueType& v) { return v.GetString(); }
    // 设置值为 const char* 数据
    static ValueType& Set(ValueType& v, const StringType data) { return v.SetString(typename ValueType::StringRefType(data)); }
    // 设置值为 const char* 数据，使用指定的分配器
    static ValueType& Set(ValueType& v, const StringType data, typename ValueType::AllocatorType& a) { return v.SetString(data, a); }
};

// 如果定义了 RAPIDJSON_HAS_STDSTRING 宏，则处理 std::basic_string 类型的值
#if RAPIDJSON_HAS_STDSTRING
template<typename ValueType>
struct TypeHelper<ValueType, std::basic_string<typename ValueType::Ch> > {
    // 定义 std::basic_string 类型的别名
    typedef std::basic_string<typename ValueType::Ch> StringType;
    // 检查值是否为字符串类型
    static bool Is(const ValueType& v) { return v.IsString(); }
    // 获取值的 std::basic_string 数据
    static StringType Get(const ValueType& v) { return StringType(v.GetString(), v.GetStringLength()); }
    // 设置值为 std::basic_string 数据，使用指定的分配器
    static ValueType& Set(ValueType& v, const StringType& data, typename ValueType::AllocatorType& a) { return v.SetString(data, a); }
};
#endif

// 结构模板 TypeHelper 的特化，用于处理数组类型的值
template<typename ValueType>
struct TypeHelper<ValueType, typename ValueType::Array> {
    // 定义数组类型的别名
    typedef typename ValueType::Array ArrayType;
    // 检查值是否为数组类型
    static bool Is(const ValueType& v) { return v.IsArray(); }
    // 获取值的数组数据
    static ArrayType Get(ValueType& v) { return v.GetArray(); }
    // 设置值为数组数据
    static ValueType& Set(ValueType& v, ArrayType data) { return v = data; }
    // 设置值为数组数据，使用指定的分配器
    static ValueType& Set(ValueType& v, ArrayType data, typename ValueType::AllocatorType&) { return v = data; }
};

// 结构模板 TypeHelper 的特化，用于处理未指定类型的值
template<typename ValueType>
// 定义一个模板类 TypeHelper，用于帮助处理不同类型的值
template <typename ValueType, typename ValueType::ConstArray>
struct TypeHelper {
    typedef typename ValueType::ConstArray ArrayType;  // 定义数组类型
    static bool Is(const ValueType& v) { return v.IsArray(); }  // 判断值是否为数组类型
    static ArrayType Get(const ValueType& v) { return v.GetArray(); }  // 获取值的数组类型
};

// 定义一个模板类 TypeHelper，用于帮助处理不同类型的值
template <typename ValueType>
struct TypeHelper<ValueType, typename ValueType::Object> {
    typedef typename ValueType::Object ObjectType;  // 定义对象类型
    static bool Is(const ValueType& v) { return v.IsObject(); }  // 判断值是否为对象类型
    static ObjectType Get(ValueType& v) { return v.GetObject(); }  // 获取值的对象类型
    static ValueType& Set(ValueType& v, ObjectType data) { return v = data; }  // 设置值为对象类型
    static ValueType& Set(ValueType& v, ObjectType data, typename ValueType::AllocatorType&) { return v = data; }  // 设置值为对象类型
};

// 定义一个模板类 TypeHelper，用于帮助处理不同类型的值
template <typename ValueType>
struct TypeHelper<ValueType, typename ValueType::ConstObject> {
    typedef typename ValueType::ConstObject ObjectType;  // 定义常量对象类型
    static bool Is(const ValueType& v) { return v.IsObject(); }  // 判断值是否为对象类型
    static ObjectType Get(const ValueType& v) { return v.GetObject(); }  // 获取值的常量对象类型
};

} // namespace internal

// 前向声明
template <bool, typename> class GenericArray;  // 声明 GenericArray 模板类
template <bool, typename> class GenericObject;  // 声明 GenericObject 模板类

///////////////////////////////////////////////////////////////////////////////
// GenericValue

//! 表示一个 JSON 值。使用 Value 用于 UTF8 编码和默认分配器。
/*!
    一个 JSON 值可以是 7 种类型之一。这个类是支持这些类型的变体类型。

    使用 Value 用于 UTF8 编码和默认分配器

    \tparam Encoding    值的编码。（即使非字符串值也需要在文档中具有相同的编码）
    \tparam Allocator   用于分配对象、数组和字符串内存的分配器类型。
*/
template <typename Encoding, typename Allocator = RAPIDJSON_DEFAULT_ALLOCATOR >
class GenericValue {
public:
    //! 对象中的名称-值对。
    typedef GenericMember<Encoding, Allocator> Member;  // 定义成员类型
    typedef Encoding EncodingType;                  //!< 从模板参数中获取的编码类型。
    // 从模板参数中获取分配器类型
    typedef Allocator AllocatorType;                
    // 从编码中获取字符类型
    typedef typename Encoding::Ch Ch;               
    // 引用常量字符串的类型
    typedef GenericStringRef<Ch> StringRefType;     
    // 用于在对象中进行迭代的成员迭代器
    typedef typename GenericMemberIterator<false,Encoding,Allocator>::Iterator MemberIterator;  
    // 用于在对象中进行常量迭代的成员迭代器
    typedef typename GenericMemberIterator<true,Encoding,Allocator>::Iterator ConstMemberIterator;  
    // 用于在数组中进行迭代的值迭代器
    typedef GenericValue* ValueIterator;            
    // 用于在数组中进行常量迭代的值迭代器
    typedef const GenericValue* ConstValueIterator; 
    // 自身的值类型
    typedef GenericValue<Encoding, Allocator> ValueType;    
    // 非常量数组类型
    typedef GenericArray<false, ValueType> Array;
    // 常量数组类型
    typedef GenericArray<true, ValueType> ConstArray;
    // 非常量对象类型
    typedef GenericObject<false, ValueType> Object;
    // 常量对象类型
    typedef GenericObject<true, ValueType> ConstObject;

    //!@name Constructors and destructor.
    //@{

    // 默认构造函数创建一个空值
    GenericValue() RAPIDJSON_NOEXCEPT : data_() { data_.f.flags = kNullFlag; }
#if RAPIDJSON_HAS_CXX11_RVALUE_REFS
    //! 如果支持 C++11 的右值引用，则定义移动构造函数
    GenericValue(GenericValue&& rhs) RAPIDJSON_NOEXCEPT : data_(rhs.data_) {
        // 将原对象的数据指针置为空，放弃原对象的内容
        rhs.data_.f.flags = kNullFlag; // give up contents
    }
#endif

private:
    //! 不允许使用复制构造函数
    GenericValue(const GenericValue& rhs);

#if RAPIDJSON_HAS_CXX11_RVALUE_REFS
    //! 不允许从 GenericDocument 移动构造
    template <typename StackAllocator>
    GenericValue(GenericDocument<Encoding,Allocator,StackAllocator>&& rhs);

    //! 不允许从 GenericDocument 移动赋值
    template <typename StackAllocator>
    GenericValue& operator=(GenericDocument<Encoding,Allocator,StackAllocator>&& rhs);
#endif

public:

    //! 使用 JSON 值类型进行构造
    /*! 创建指定类型的值，并使用默认内容。
        \param type 值的类型
        \note 数字类型的默认内容为零。
    */
    explicit GenericValue(Type type) RAPIDJSON_NOEXCEPT : data_() {
        static const uint16_t defaultFlags[] = {
            kNullFlag, kFalseFlag, kTrueFlag, kObjectFlag, kArrayFlag, kShortStringFlag,
            kNumberAnyFlag
        };
        RAPIDJSON_NOEXCEPT_ASSERT(type >= kNullType && type <= kNumberType);
        data_.f.flags = defaultFlags[type];

        // 使用 ShortString 存储空字符串
        if (type == kStringType)
            data_.ss.SetLength(0);
    }

    //! 显式的复制构造函数（带有分配器）
    /*! 通过给定的分配器创建值的副本
        \tparam SourceAllocator \c rhs 的分配器
        \param rhs 要复制的值（只读）
        \param allocator 用于分配复制元素和缓冲区的分配器。通常使用 GenericDocument::GetAllocator()。
        \param copyConstStrings 强制复制常量字符串（例如引用原地缓冲区）
        \see CopyFrom()
    */
    template <typename SourceAllocator>
    // 根据传入的 rhs 对象和 allocator 创建一个新的 GenericValue 对象
    GenericValue(const GenericValue<Encoding,SourceAllocator>& rhs, Allocator& allocator, bool copyConstStrings = false) {
        // 根据 rhs 对象的类型进行不同的处理
        switch (rhs.GetType()) {
        // 如果是对象类型
        case kObjectType:
            // 调用 DoCopyMembers 方法复制成员
            DoCopyMembers(rhs, allocator, copyConstStrings);
            break;
        // 如果是数组类型
        case kArrayType: {
                // 获取数组元素个数
                SizeType count = rhs.data_.a.size;
                // 在 allocator 上分配 count 个 GenericValue 对象的内存
                GenericValue* le = reinterpret_cast<GenericValue*>(allocator.Malloc(count * sizeof(GenericValue)));
                // 获取 rhs 的元素指针
                const GenericValue<Encoding,SourceAllocator>* re = rhs.GetElementsPointer();
                // 遍历 rhs 的元素，调用拷贝构造函数创建新的 GenericValue 对象
                for (SizeType i = 0; i < count; i++)
                    new (&le[i]) GenericValue(re[i], allocator, copyConstStrings);
                // 设置标记为数组类型，并设置大小和容量
                data_.f.flags = kArrayFlag;
                data_.a.size = data_.a.capacity = count;
                // 设置元素指针
                SetElementsPointer(le);
            }
            break;
        // 如果是字符串类型
        case kStringType:
            // 如果 rhs 是常量字符串并且不需要复制常量字符串
            if (rhs.data_.f.flags == kConstStringFlag && !copyConstStrings) {
                // 直接复制标记和数据
                data_.f.flags = rhs.data_.f.flags;
                data_  = *reinterpret_cast<const Data*>(&rhs.data_);
            }
            else
                // 否则调用 SetStringRaw 方法设置字符串
                SetStringRaw(StringRef(rhs.GetString(), rhs.GetStringLength()), allocator);
            break;
        // 其他类型
        default:
            // 直接复制标记和数据
            data_.f.flags = rhs.data_.f.flags;
            data_  = *reinterpret_cast<const Data*>(&rhs.data_);
            break;
        }
    }

    //! Constructor for boolean value.
    /*! \param b Boolean value
        \note This constructor is limited to \em real boolean values and rejects
            implicitly converted types like arbitrary pointers.  Use an explicit cast
            to \c bool, if you want to construct a boolean JSON value in such cases.
     */
#ifndef RAPIDJSON_DOXYGEN_RUNNING // hide SFINAE from Doxygen
    // 如果不是在 Doxygen 运行时，根据传入的 bool 类型参数创建 GenericValue 对象
    template <typename T>
    explicit GenericValue(T b, RAPIDJSON_ENABLEIF((internal::IsSame<bool, T>))) RAPIDJSON_NOEXCEPT  // See #472
#else
    // 如果在 Doxygen 运行时，根据传入的 bool 类型参数创建 GenericValue 对象
    explicit GenericValue(bool b) RAPIDJSON_NOEXCEPT
#endif
        : data_() {
            // 防止 SFINAE 失效
            RAPIDJSON_STATIC_ASSERT((internal::IsSame<bool,T>::Value));
            // 根据传入的 bool 值设置 GenericValue 对象的标志位
            data_.f.flags = b ? kTrueFlag : kFalseFlag;
    }

    //! Constructor for int value.
    // 根据传入的 int 类型参数创建 GenericValue 对象
    explicit GenericValue(int i) RAPIDJSON_NOEXCEPT : data_() {
        // 将传入的 int 值赋给 GenericValue 对象的数据成员
        data_.n.i64 = i;
        // 根据传入的 int 值设置 GenericValue 对象的标志位
        data_.f.flags = (i >= 0) ? (kNumberIntFlag | kUintFlag | kUint64Flag) : kNumberIntFlag;
    }

    //! Constructor for unsigned value.
    // 根据传入的 unsigned 类型参数创建 GenericValue 对象
    explicit GenericValue(unsigned u) RAPIDJSON_NOEXCEPT : data_() {
        // 将传入的 unsigned 值赋给 GenericValue 对象的数据成员
        data_.n.u64 = u;
        // 根据传入的 unsigned 值设置 GenericValue 对象的标志位
        data_.f.flags = (u & 0x80000000) ? kNumberUintFlag : (kNumberUintFlag | kIntFlag | kInt64Flag);
    }

    //! Constructor for int64_t value.
    // 根据传入的 int64_t 类型参数创建 GenericValue 对象
    explicit GenericValue(int64_t i64) RAPIDJSON_NOEXCEPT : data_() {
        // 将传入的 int64_t 值赋给 GenericValue 对象的数据成员
        data_.n.i64 = i64;
        // 根据传入的 int64_t 值设置 GenericValue 对象的标志位
        data_.f.flags = kNumberInt64Flag;
        if (i64 >= 0) {
            data_.f.flags |= kNumberUint64Flag;
            if (!(static_cast<uint64_t>(i64) & RAPIDJSON_UINT64_C2(0xFFFFFFFF, 0x00000000)))
                data_.f.flags |= kUintFlag;
            if (!(static_cast<uint64_t>(i64) & RAPIDJSON_UINT64_C2(0xFFFFFFFF, 0x80000000)))
                data_.f.flags |= kIntFlag;
        }
        else if (i64 >= static_cast<int64_t>(RAPIDJSON_UINT64_C2(0xFFFFFFFF, 0x80000000)))
            data_.f.flags |= kIntFlag;
    }

    //! Constructor for uint64_t value.
    // 根据传入的 uint64_t 类型参数创建 GenericValue 对象
    // 以无符号64位整数构造函数，初始化数据成员
    explicit GenericValue(uint64_t u64) RAPIDJSON_NOEXCEPT : data_() {
        // 将无符号64位整数赋值给数据成员
        data_.n.u64 = u64;
        // 设置数据成员的标志位为kNumberUint64Flag
        data_.f.flags = kNumberUint64Flag;
        // 如果u64不包含最高位为1的情况，设置数据成员的标志位为kInt64Flag
        if (!(u64 & RAPIDJSON_UINT64_C2(0x80000000, 0x00000000)))
            data_.f.flags |= kInt64Flag;
        // 如果u64不包含最高位为1的情况，设置数据成员的标志位为kUintFlag
        if (!(u64 & RAPIDJSON_UINT64_C2(0xFFFFFFFF, 0x00000000)))
            data_.f.flags |= kUintFlag;
        // 如果u64不包含最高位为1的情况，设置数据成员的标志位为kIntFlag
        if (!(u64 & RAPIDJSON_UINT64_C2(0xFFFFFFFF, 0x80000000)))
            data_.f.flags |= kIntFlag;
    }

    //! 双精度浮点数构造函数
    explicit GenericValue(double d) RAPIDJSON_NOEXCEPT : data_() { data_.n.d = d; data_.f.flags = kNumberDoubleFlag; }

    //! 单精度浮点数构造函数
    explicit GenericValue(float f) RAPIDJSON_NOEXCEPT : data_() { data_.n.d = static_cast<double>(f); data_.f.flags = kNumberDoubleFlag; }

    //! 常量字符串构造函数（不复制字符串）
    GenericValue(const Ch* s, SizeType length) RAPIDJSON_NOEXCEPT : data_() { SetStringRaw(StringRef(s, length)); }

    //! 常量字符串构造函数（不复制字符串）
    explicit GenericValue(StringRefType s) RAPIDJSON_NOEXCEPT : data_() { SetStringRaw(s); }

    //! 复制字符串构造函数（复制字符串）
    GenericValue(const Ch* s, SizeType length, Allocator& allocator) : data_() { SetStringRaw(StringRef(s, length), allocator); }

    //! 复制字符串构造函数（复制字符串）
    GenericValue(const Ch*s, Allocator& allocator) : data_() { SetStringRaw(StringRef(s), allocator); }
#if RAPIDJSON_HAS_STDSTRING
    //! 如果定义了 RAPIDJSON_HAS_STDSTRING 预处理符号，则使用该构造函数从字符串对象中复制字符串（即复制字符串）
    /*! \note 需要定义预处理符号 \ref RAPIDJSON_HAS_STDSTRING。
     */
    GenericValue(const std::basic_string<Ch>& s, Allocator& allocator) : data_() { SetStringRaw(StringRef(s), allocator); }
#endif

    //! 用于数组的构造函数。
    /*!
        \param a 通过 \c GetArray() 获得的数组。
        \note \c Array 总是按值传递。
        \note 源数组被移动到该值中，源数组变为空。
    */
    GenericValue(Array a) RAPIDJSON_NOEXCEPT : data_(a.value_.data_) {
        a.value_.data_ = Data();
        a.value_.data_.f.flags = kArrayFlag;
    }

    //! 用于对象的构造函数。
    /*!
        \param o 通过 \c GetObject() 获得的对象。
        \note \c Object 总是按值传递。
        \note 源对象被移动到该值中，源对象变为空。
    */
    GenericValue(Object o) RAPIDJSON_NOEXCEPT : data_(o.value_.data_) {
        o.value_.data_ = Data();
        o.value_.data_.f.flags = kObjectFlag;
    }

    //! 析构函数。
    /*! 需要销毁数组的元素，对象的成员，或复制的字符串。
    */
    // 析构函数，用于释放内存
    ~GenericValue() {
        // 如果需要释放内存或者使用了成员映射，并且分配器是引用计数的（例如 MemoryPoolAllocator）
        if (Allocator::kNeedFree || (RAPIDJSON_USE_MEMBERSMAP+0 &&
                                     internal::IsRefCounted<Allocator>::Value)) {
            // 根据值的类型进行不同的处理
            switch(data_.f.flags) {
            // 如果是数组类型
            case kArrayFlag:
                {
                    // 获取数组元素的指针
                    GenericValue* e = GetElementsPointer();
                    // 遍历数组元素，调用析构函数释放内存
                    for (GenericValue* v = e; v != e + data_.a.size; ++v)
                        v->~GenericValue();
                    // 如果需要释放内存，直接调用分配器的 Free 方法
                    if (Allocator::kNeedFree) { // Shortcut by Allocator's trait
                        Allocator::Free(e);
                    }
                }
                break;
            // 如果是对象类型
            case kObjectFlag:
                // 调用 DoFreeMembers 方法释放内存
                DoFreeMembers();
                break;
            // 如果是拷贝字符串类型
            case kCopyStringFlag:
                // 如果需要释放内存，直接调用分配器的 Free 方法
                if (Allocator::kNeedFree) { // Shortcut by Allocator's trait
                    Allocator::Free(const_cast<Ch*>(GetStringPointer()));
                }
                break;
            // 其他类型不做处理
            default:
                break;  // Do nothing for other types.
            }
        }
    }

    //@}

    //!@name 赋值运算符
    //@{

    //! 使用移动语义进行赋值
    /*! \param rhs 赋值的来源。在赋值后，它将变成一个空值。
    */
    GenericValue& operator=(GenericValue& rhs) RAPIDJSON_NOEXCEPT {
        // 如果不是自我赋值
        if (RAPIDJSON_LIKELY(this != &rhs)) {
            // 在赋值 "rhs" 之前不能销毁 "this"，否则如果 "rhs" 是 "this" 的子值，那么在释放后 "rhs" 可能会被使用，因此需要进行临时处理
            GenericValue temp;
            temp.RawAssign(rhs);
            this->~GenericValue();
            RawAssign(temp);
        }
        return *this;
    }
#if RAPIDJSON_HAS_CXX11_RVALUE_REFS
    //! 如果支持 C++11 的右值引用，则进行移动赋值
    GenericValue& operator=(GenericValue&& rhs) RAPIDJSON_NOEXCEPT {
        return *this = rhs.Move();
    }
#endif

    //! 使用常量字符串引用进行赋值（无需复制）
    /*! \param str 要赋值的常量字符串引用
        \note 需要这个重载是为了避免与下面的通用基本类型赋值重载发生冲突。
        \see GenericStringRef, operator=(T)
    */
    GenericValue& operator=(StringRefType str) RAPIDJSON_NOEXCEPT {
        GenericValue s(str);
        return *this = s;
    }

    //! 使用基本类型进行赋值
    /*! \tparam T 可以是 \ref Type, \c int, \c unsigned, \c int64_t, \c uint64_t
        \param value 要赋值的值

        \note 源类型 \c T 明确禁止所有指针类型，特别是 (\c const) \ref Ch*。这有助于避免隐式引用具有不足寿命的字符字符串，使用 \ref SetString(const Ch*, Allocator&)（用于复制）或 \ref StringRef()（显式标记指针为常量）来代替。
        所有其他指针类型将隐式转换为 \c bool，使用 \ref SetBool() 代替。
    */
    template <typename T>
    RAPIDJSON_DISABLEIF_RETURN((internal::IsPointer<T>), (GenericValue&))
    operator=(T value) {
        GenericValue v(value);
        return *this = v;
    }

    //! 从 Value 进行深拷贝赋值
    /*! 将 Value 的 \b 拷贝赋值给当前的 Value 对象
        \tparam SourceAllocator \c rhs 的分配器类型
        \param rhs 要从中拷贝的 Value（只读）
        \param allocator 用于拷贝的分配器
        \param copyConstStrings 强制复制常量字符串（例如引用原位缓冲区）
     */
    template <typename SourceAllocator>
    // 从另一个 GenericValue 对象复制内容到当前对象，使用指定的分配器，可以选择是否复制常量字符串
    GenericValue& CopyFrom(const GenericValue<Encoding, SourceAllocator>& rhs, Allocator& allocator, bool copyConstStrings = false) {
        // 断言当前对象和另一个对象不是同一个对象
        RAPIDJSON_ASSERT(static_cast<void*>(this) != static_cast<void const*>(&rhs));
        // 调用当前对象的析构函数
        this->~GenericValue();
        // 在当前对象上构造一个新的 GenericValue 对象，使用另一个对象的内容和指定的分配器
        new (this) GenericValue(rhs, allocator, copyConstStrings);
        // 返回当前对象的引用
        return *this;
    }

    //! 交换当前对象的内容和另一个对象的内容
    /*!
        \param other 另一个值
        \note 常量复杂度
    */
    GenericValue& Swap(GenericValue& other) RAPIDJSON_NOEXCEPT {
        // 创建一个临时的 GenericValue 对象
        GenericValue temp;
        // 将当前对象的内容赋值给临时对象
        temp.RawAssign(*this);
        // 将另一个对象的内容赋值给当前对象
        RawAssign(other);
        // 将临时对象的内容赋值给另一个对象
        other.RawAssign(temp);
        // 返回当前对象的引用
        return *this;
    }

    //! 自由交换函数助手
    /*!
        用于支持基于 \c std::swap 的常见交换实现模式的辅助函数
        \see Swap()
     */
    // 定义一个友元函数 swap，调用当前对象的 Swap 方法
    friend inline void swap(GenericValue& a, GenericValue& b) RAPIDJSON_NOEXCEPT { a.Swap(b); }

    //! 准备 Value 以支持移动语义
    /*! \return *this */
    GenericValue& Move() RAPIDJSON_NOEXCEPT { return *this; }
    //@}

    //!@name 等于和不等于操作符
    //@{
    //! 等于操作符
    /*!
        \note 如果对象包含重复的命名成员，则与任何对象的相等比较始终为 \c false。
        \note 复杂度对于对象的成员数量是二次的，对于其余部分是线性的（子树中所有值的数量和所有字符串的总长度）。
    */
    template <typename SourceAllocator>
    // 重载等于运算符，用于比较两个 GenericValue 对象是否相等
    bool operator==(const GenericValue<Encoding, SourceAllocator>& rhs) const {
        // 定义一个类型别名 RhsType，指向 GenericValue<Encoding, SourceAllocator>
        typedef GenericValue<Encoding, SourceAllocator> RhsType;
        // 如果当前对象的类型与 rhs 对象的类型不相同，则返回 false
        if (GetType() != rhs.GetType())
            return false;

        // 根据当前对象的类型进行不同的比较
        switch (GetType()) {
        case kObjectType: // Warning: O(n^2) inner-loop
            // 如果对象类型为 kObjectType，比较对象的成员数量，不相同则返回 false
            if (data_.o.size != rhs.data_.o.size)
                return false;
            // 遍历当前对象的成员，查找对应的 rhs 对象的成员进行比较
            for (ConstMemberIterator lhsMemberItr = MemberBegin(); lhsMemberItr != MemberEnd(); ++lhsMemberItr) {
                // 在 rhs 对象中查找当前成员的迭代器
                typename RhsType::ConstMemberIterator rhsMemberItr = rhs.FindMember(lhsMemberItr->name);
                // 如果在 rhs 对象中未找到对应的成员，或者对应的值不相等，则返回 false
                if (rhsMemberItr == rhs.MemberEnd() || lhsMemberItr->value != rhsMemberItr->value)
                    return false;
            }
            // 所有成员比较完毕，返回 true
            return true;

        case kArrayType:
            // 如果对象类型为 kArrayType，比较数组的大小，不相同则返回 false
            if (data_.a.size != rhs.data_.a.size)
                return false;
            // 遍历数组元素，逐个比较对应位置的元素
            for (SizeType i = 0; i < data_.a.size; i++)
                // 如果当前对象的第 i 个元素与 rhs 对象的第 i 个元素不相等，则返回 false
                if ((*this)[i] != rhs[i])
                    return false;
            // 所有元素比较完毕，返回 true
            return true;

        case kStringType:
            // 如果对象类型为 kStringType，调用 StringEqual 方法进行比较
            return StringEqual(rhs);

        case kNumberType:
            // 如果对象类型为 kNumberType
            if (IsDouble() || rhs.IsDouble()) {
                double a = GetDouble();     // May convert from integer to double.
                double b = rhs.GetDouble(); // Ditto
                // 如果当前对象或 rhs 对象为 double 类型，则比较两个 double 值是否相等
                return a >= b && a <= b;    // Prevent -Wfloat-equal
            }
            else
                // 否则比较两个整数值是否相等
                return data_.n.u64 == rhs.data_.n.u64;

        default:
            // 其他类型直接返回 true
            return true;
        }
    }

    //! Equal-to operator with const C-string pointer
    // 重载等于运算符，用于比较当前对象与常量 C 字符串指针是否相等
    bool operator==(const Ch* rhs) const { return *this == GenericValue(StringRef(rhs)); }
#if RAPIDJSON_HAS_STDSTRING
    //! Equal-to operator with string object
    /*! \note Requires the definition of the preprocessor symbol \ref RAPIDJSON_HAS_STDSTRING.
     */
    bool operator==(const std::basic_string<Ch>& rhs) const { return *this == GenericValue(StringRef(rhs)); }
#endif

    //! Equal-to operator with primitive types
    /*! \tparam T Either \ref Type, \c int, \c unsigned, \c int64_t, \c uint64_t, \c double, \c true, \c false
    */
    template <typename T> RAPIDJSON_DISABLEIF_RETURN((internal::OrExpr<internal::IsPointer<T>,internal::IsGenericValue<T> >), (bool)) operator==(const T& rhs) const { return *this == GenericValue(rhs); }

    //! Not-equal-to operator
    /*! \return !(*this == rhs)
     */
    template <typename SourceAllocator>
    bool operator!=(const GenericValue<Encoding, SourceAllocator>& rhs) const { return !(*this == rhs); }

    //! Not-equal-to operator with const C-string pointer
    bool operator!=(const Ch* rhs) const { return !(*this == rhs); }

    //! Not-equal-to operator with arbitrary types
    /*! \return !(*this == rhs)
     */
    template <typename T> RAPIDJSON_DISABLEIF_RETURN((internal::IsGenericValue<T>), (bool)) operator!=(const T& rhs) const { return !(*this == rhs); }

#ifndef __cpp_lib_three_way_comparison
    //! Equal-to operator with arbitrary types (symmetric version)
    /*! \return (rhs == lhs)
     */
    template <typename T> friend RAPIDJSON_DISABLEIF_RETURN((internal::IsGenericValue<T>), (bool)) operator==(const T& lhs, const GenericValue& rhs) { return rhs == lhs; }

    //! Not-Equal-to operator with arbitrary types (symmetric version)
    /*! \return !(rhs == lhs)
     */
    template <typename T> friend RAPIDJSON_DISABLEIF_RETURN((internal::IsGenericValue<T>), (bool)) operator!=(const T& lhs, const GenericValue& rhs) { return !(rhs == lhs); }
    //@}
#endif

    //!@name Type
    //@{

    Type GetType()  const { return static_cast<Type>(data_.f.flags & kTypeMask); }
    // 检查当前数据是否为空值
    bool IsNull()   const { return data_.f.flags == kNullFlag; }
    // 检查当前数据是否为假值
    bool IsFalse()  const { return data_.f.flags == kFalseFlag; }
    // 检查当前数据是否为真值
    bool IsTrue()   const { return data_.f.flags == kTrueFlag; }
    // 检查当前数据是否为布尔类型
    bool IsBool()   const { return (data_.f.flags & kBoolFlag) != 0; }
    // 检查当前数据是否为对象
    bool IsObject() const { return data_.f.flags == kObjectFlag; }
    // 检查当前数据是否为数组
    bool IsArray()  const { return data_.f.flags == kArrayFlag; }
    // 检查当前数据是否为数字类型
    bool IsNumber() const { return (data_.f.flags & kNumberFlag) != 0; }
    // 检查当前数据是否为整数
    bool IsInt()    const { return (data_.f.flags & kIntFlag) != 0; }
    // 检查当前数据是否为无符号整数
    bool IsUint()   const { return (data_.f.flags & kUintFlag) != 0; }
    // 检查当前数据是否为64位整数
    bool IsInt64()  const { return (data_.f.flags & kInt64Flag) != 0; }
    // 检查当前数据是否为64位无符号整数
    bool IsUint64() const { return (data_.f.flags & kUint64Flag) != 0; }
    // 检查当前数据是否为双精度浮点数
    bool IsDouble() const { return (data_.f.flags & kDoubleFlag) != 0; }
    // 检查当前数据是否为字符串
    bool IsString() const { return (data_.f.flags & kStringFlag) != 0; }

    // 检查一个数字是否可以无损地转换为双精度浮点数
    bool IsLosslessDouble() const {
        // 如果当前数据不是数字类型，则返回false
        if (!IsNumber()) return false;
        // 如果是64位无符号整数
        if (IsUint64()) {
            uint64_t u = GetUint64();
            // 使用volatile关键字声明的变量，防止编译器优化
            volatile double d = static_cast<double>(u);
            // 检查是否可以无损转换为双精度浮点数
            return (d >= 0.0)
                && (d < static_cast<double>((std::numeric_limits<uint64_t>::max)()))
                && (u == static_cast<uint64_t>(d));
        }
        // 如果是64位整数
        if (IsInt64()) {
            int64_t i = GetInt64();
            // 使用volatile关键字声明的变量，防止编译器优化
            volatile double d = static_cast<double>(i);
            // 检查是否可以无损转换为双精度浮点数
            return (d >= static_cast<double>((std::numeric_limits<int64_t>::min)()))
                && (d < static_cast<double>((std::numeric_limits<int64_t>::max)()))
                && (i == static_cast<int64_t>(d));
        }
        // 其他情况均返回true
        return true; // double, int, uint are always lossless
    }

    // 检查一个数字是否为浮点数（可能有损失）
    // 检查当前值是否为浮点数
    bool IsFloat() const  {
        // 如果当前值不是双精度浮点数，则返回 false
        if ((data_.f.flags & kDoubleFlag) == 0)
            return false;
        // 获取当前值的双精度浮点数表示
        double d = GetDouble();
        // 检查双精度浮点数是否在单精度浮点数范围内
        return d >= -3.4028234e38 && d <= 3.4028234e38;
    }
    // 检查一个数字是否可以无损地转换为浮点数
    bool IsLosslessFloat() const {
        // 如果当前值不是数字，则返回 false
        if (!IsNumber()) return false;
        // 获取当前值的双精度浮点数表示
        double a = GetDouble();
        // 检查双精度浮点数是否可以转换为单精度浮点数
        if (a < static_cast<double>(-(std::numeric_limits<float>::max)())
                || a > static_cast<double>((std::numeric_limits<float>::max)()))
            return false;
        // 将双精度浮点数转换为单精度浮点数，并检查是否有损失
        double b = static_cast<double>(static_cast<float>(a));
        return a >= b && a <= b;    // 防止 -Wfloat-equal
    }

    //@}

    //!@name Null
    //@{

    // 将当前值设置为 null
    GenericValue& SetNull() { this->~GenericValue(); new (this) GenericValue(); return *this; }

    //@}

    //!@name Bool
    //@{

    // 获取当前值的布尔值
    bool GetBool() const { RAPIDJSON_ASSERT(IsBool()); return data_.f.flags == kTrueFlag; }
    //!< 设置布尔值
    /*! \post IsBool() == true */
    // 设置当前值为布尔值
    GenericValue& SetBool(bool b) { this->~GenericValue(); new (this) GenericValue(b); return *this; }

    //@}

    //!@name Object
    //@{

    //! 将当前值设置为空对象
    /*! \post IsObject() == true */
    // 设置当前值为空对象
    GenericValue& SetObject() { this->~GenericValue(); new (this) GenericValue(kObjectType); return *this; }

    //! 获取对象中成员的数量
    SizeType MemberCount() const { RAPIDJSON_ASSERT(IsObject()); return data_.o.size; }

    //! 获取对象的容量
    SizeType MemberCapacity() const { RAPIDJSON_ASSERT(IsObject()); return data_.o.capacity; }

    //! 检查对象是否为空
    bool ObjectEmpty() const { RAPIDJSON_ASSERT(IsObject()); return data_.o.size == 0; }

    //! 从对象中获取与给定名称关联的值
    /*! \pre IsObject() == true
        \tparam T Either \c Ch or \c const \c Ch (template used for disambiguation with \ref operator[](SizeType))
        \note In version 0.1x, if the member is not found, this function returns a null value. This makes issue 7.
        Since 0.2, if the name is not correct, it will assert.
        If user is unsure whether a member exists, user should use HasMember() first.
        A better approach is to use FindMember().
        \note Linear time complexity.
    */
    // 定义模板函数，根据名称获取对象中的值
    template <typename T>
    RAPIDJSON_DISABLEIF_RETURN((internal::NotExpr<internal::IsSame<typename internal::RemoveConst<T>::Type, Ch> >),(GenericValue&)) operator[](T* name) {
        // 创建一个字符串引用对象
        GenericValue n(StringRef(name));
        // 调用重载的 operator[] 函数
        return (*this)[n];
    }
    // 定义模板函数，根据名称获取对象中的值（常量版本）
    template <typename T>
    RAPIDJSON_DISABLEIF_RETURN((internal::NotExpr<internal::IsSame<typename internal::RemoveConst<T>::Type, Ch> >),(const GenericValue&)) operator[](T* name) const { return const_cast<GenericValue&>(*this)[name]; }

    //! Get a value from an object associated with the name.
    /*! \pre IsObject() == true
        \tparam SourceAllocator Allocator of the \c name value

        \note Compared to \ref operator[](T*), this version is faster because it does not need a StrLen().
        And it can also handle strings with embedded null characters.

        \note Linear time complexity.
    */
    // 定义模板函数，根据名称获取对象中的值，使用指定的分配器
    template <typename SourceAllocator>
    // 重载操作符[]，用于通过名称获取成员值
    GenericValue& operator[](const GenericValue<Encoding, SourceAllocator>& name) {
        // 查找指定名称的成员
        MemberIterator member = FindMember(name);
        // 如果找到了成员，则返回其值
        if (member != MemberEnd())
            return member->value;
        // 如果未找到成员
        else {
            RAPIDJSON_ASSERT(false);    // see above note

            // This will generate -Wexit-time-destructors in clang
            // static GenericValue NullValue;
            // return NullValue;

            // Use static buffer and placement-new to prevent destruction
            // 使用静态缓冲区和放置new来防止销毁
            static char buffer[sizeof(GenericValue)];
            // 在静态缓冲区上放置新的GenericValue对象，并返回其引用
            return *new (buffer) GenericValue();
        }
    }
    // 重载操作符[]，用于通过名称获取成员值（常量版本）
    template <typename SourceAllocator>
    const GenericValue& operator[](const GenericValue<Encoding, SourceAllocator>& name) const { return const_cast<GenericValue&>(*this)[name]; }
#if RAPIDJSON_HAS_STDSTRING
    //! 如果编译时启用了对 std::string 的支持，则定义一个从对象中获取值的操作符，使用字符串对象作为键
    GenericValue& operator[](const std::basic_string<Ch>& name) { return (*this)[GenericValue(StringRef(name))]; }
    //! 如果编译时启用了对 std::string 的支持，则定义一个从对象中获取值的操作符，使用字符串对象作为键
    const GenericValue& operator[](const std::basic_string<Ch>& name) const { return (*this)[GenericValue(StringRef(name))]; }
#endif

    //! 返回对象的常量成员迭代器
    /*! \pre IsObject() == true */
    ConstMemberIterator MemberBegin() const { RAPIDJSON_ASSERT(IsObject()); return ConstMemberIterator(GetMembersPointer()); }
    //! 返回对象的常量成员迭代器的末尾
    /*! \pre IsObject() == true */
    ConstMemberIterator MemberEnd() const   { RAPIDJSON_ASSERT(IsObject()); return ConstMemberIterator(GetMembersPointer() + data_.o.size); }
    //! 返回对象的成员迭代器
    /*! \pre IsObject() == true */
    MemberIterator MemberBegin()            { RAPIDJSON_ASSERT(IsObject()); return MemberIterator(GetMembersPointer()); }
    //! 返回对象的成员迭代器的末尾
    /*! \pre IsObject() == true */
    MemberIterator MemberEnd()              { RAPIDJSON_ASSERT(IsObject()); return MemberIterator(GetMembersPointer() + data_.o.size); }

    //! 请求对象具有足够的容量来存储成员
    /*! \param newCapacity  对象至少需要具有的容量
        \param allocator    重新分配内存的分配器。必须与之前使用的相同。通常使用 GenericDocument::GetAllocator()。
        \return 用于流畅 API 的值本身。
        \note 线性时间复杂度。
    */
    GenericValue& MemberReserve(SizeType newCapacity, Allocator &allocator) {
        RAPIDJSON_ASSERT(IsObject());
        DoReserveMembers(newCapacity, allocator);
        return *this;
    }

    //! 检查对象中是否存在成员
    /*!
        \param name 要搜索的成员名称。
        \pre IsObject() == true  // 先决条件：确保当前对象是一个 JSON 对象
        \return 是否存在具有该名称的成员。
        \note 如果需要获取值，最好直接使用 FindMember()。
        \note 线性时间复杂度。
    */
    bool HasMember(const Ch* name) const { return FindMember(name) != MemberEnd(); }  // 返回是否存在具有给定名称的成员
#if RAPIDJSON_HAS_STDSTRING
    //! 如果编译器支持 std::string，则检查对象中是否存在指定成员
    /*!
        \param name 要搜索的成员名称
        \pre IsObject() == true
        \return 是否存在具有该名称的成员
        \note 如果需要获取值，最好直接使用 FindMember()
        \note 线性时间复杂度
    */
    bool HasMember(const std::basic_string<Ch>& name) const { return FindMember(name) != MemberEnd(); }
#endif

    //! 使用 GenericValue 名称检查对象中是否存在成员
    /*!
        这个版本更快，因为它不需要 StrLen()。它还可以处理带有空字符的字符串。
        \param name 要搜索的成员名称
        \pre IsObject() == true
        \return 是否存在具有该名称的成员
        \note 如果需要获取值，最好直接使用 FindMember()
        \note 线性时间复杂度
    */
    template <typename SourceAllocator>
    bool HasMember(const GenericValue<Encoding, SourceAllocator>& name) const { return FindMember(name) != MemberEnd(); }

    //! 通过名称查找成员
    /*!
        \param name 要搜索的成员名称
        \pre IsObject() == true
        \return 成员的迭代器，如果存在的话
            否则返回 \ref MemberEnd()。

        \note Rapidjson 的早期版本返回一个 \c NULL 指针，如果请求的成员不存在。为了与例如
            \c std::map 的一致性，现在已经更改为返回 MemberEnd()。
        \note 线性时间复杂度
    */
    MemberIterator FindMember(const Ch* name) {
        GenericValue n(StringRef(name));
        return FindMember(n);
    }

    ConstMemberIterator FindMember(const Ch* name) const { return const_cast<GenericValue&>(*this).FindMember(name); }

    //! 通过名称查找成员
    /*!
        这个版本更快，因为它不需要 StrLen()。它还可以处理带有空字符的字符串。
        \param name 要搜索的成员名称。
        \pre IsObject() == true
        \return 如果存在，则返回成员的迭代器。否则返回 \ref MemberEnd()。

        \note Rapidjson 的早期版本在请求的成员不存在时返回一个 \c NULL 指针。为了与例如 \c std::map 的一致性，现在已经改为返回 MemberEnd()。
        \note 线性时间复杂度。
    */
    template <typename SourceAllocator>
    MemberIterator FindMember(const GenericValue<Encoding, SourceAllocator>& name) {
        RAPIDJSON_ASSERT(IsObject());  // 断言对象类型为对象
        RAPIDJSON_ASSERT(name.IsString());  // 断言传入的名称为字符串类型
        return DoFindMember(name);  // 调用 DoFindMember 方法查找成员
    }
    template <typename SourceAllocator> ConstMemberIterator FindMember(const GenericValue<Encoding, SourceAllocator>& name) const { return const_cast<GenericValue&>(*this).FindMember(name); }  // 返回常量成员迭代器
#if RAPIDJSON_HAS_STDSTRING
    //! 如果编译器支持 std::string，则定义以下函数

    //! 通过字符串对象名称查找成员
    /*!
        \param name 要搜索的成员名称
        \pre IsObject() == true
        \return 如果存在，则返回指向成员的迭代器。否则返回 \ref MemberEnd()。
    */
    MemberIterator FindMember(const std::basic_string<Ch>& name) { return FindMember(GenericValue(StringRef(name))); }
    ConstMemberIterator FindMember(const std::basic_string<Ch>& name) const { return FindMember(GenericValue(StringRef(name))); }
#endif

    //! 向对象添加成员（名称-值对）
    /*! \param name 作为成员名称的字符串值。
        \param value 任何类型的值。
        \param allocator 重新分配内存的分配器。必须与之前使用的相同。通常使用 GenericDocument::GetAllocator()。
        \return 用于流畅 API 的值本身。
        \note 在成功时，\c name 和 \c value 的所有权将转移到此对象。
        \pre  IsObject() && name.IsString()
        \post name.IsNull() && value.IsNull()
        \note 分摊的常数时间复杂度。
    */
    GenericValue& AddMember(GenericValue& name, GenericValue& value, Allocator& allocator) {
        RAPIDJSON_ASSERT(IsObject());
        RAPIDJSON_ASSERT(name.IsString());
        DoAddMember(name, value, allocator);
        return *this;
    }

    //! 向对象添加常量字符串值作为成员（名称-值对）
    /*! \param name 作为成员名称的字符串值。
        \param value 作为成员值的常量字符串引用。
        \param allocator 重新分配内存的分配器。必须与之前使用的相同。通常使用 GenericDocument::GetAllocator()。
        \return 用于流畅 API 的值本身。
        \pre  IsObject()
        \note 需要此重载以避免与下面的通用基本类型 AddMember(GenericValue&,T,Allocator&) 重载发生冲突。
        \note 分摊的常数时间复杂度。
    */
    # 添加成员到 JSON 对象中，成员名为 name，值为 value
    GenericValue& AddMember(GenericValue& name, StringRefType value, Allocator& allocator) {
        # 创建一个值为 value 的 GenericValue 对象
        GenericValue v(value);
        # 调用重载的 AddMember 方法，将 name 和 v 添加到 JSON 对象中
        return AddMember(name, v, allocator);
    }
#if RAPIDJSON_HAS_STDSTRING
    //! 如果编译器支持 std::string，则将字符串对象作为成员（名称-值对）添加到对象中。
    /*! \param name 成员的名称字符串值。
        \param value 作为成员值的常量字符串引用。
        \param allocator 用于重新分配内存的分配器。必须与之前使用的相同。通常使用 GenericDocument::GetAllocator()。
        \return 用于流畅 API 的值本身。
        \pre  IsObject()
        \note 需要此重载以避免与下面的通用基本类型 AddMember(GenericValue&,T,Allocator&) 重载发生冲突。
        \note 分摊的常数时间复杂度。
    */
    GenericValue& AddMember(GenericValue& name, std::basic_string<Ch>& value, Allocator& allocator) {
        GenericValue v(value, allocator);
        return AddMember(name, v, allocator);
    }
#endif

    //! 将任何原始值作为成员（名称-值对）添加到对象中。
    /*! \tparam T 要么是 \ref Type，\c int，\c unsigned，\c int64_t，\c uint64_t
        \param name 作为成员名称的字符串值。
        \param value 作为成员值的原始类型 \c T 的值
        \param allocator 用于重新分配内存的分配器。通常使用 GenericDocument::GetAllocator()。
        \return 用于流畅 API 的值本身。
        \pre  IsObject()

        \note 源类型 \c T 明确禁止所有指针类型，特别是 (\c const) \ref Ch*。这有助于避免使用寿命不足的字符字符串隐式引用，使用 \ref AddMember(StringRefType, GenericValue&, Allocator&) 或 \ref AddMember(StringRefType, StringRefType, Allocator&)。所有其他指针类型将隐式转换为 \c bool，如果需要，使用显式转换。
        \note 分摊的常数时间复杂度。
    */
    template <typename T>
    # 如果 T 是指针类型或者是 GenericValue 类型，则禁用该函数并返回 GenericValue 的引用
    RAPIDJSON_DISABLEIF_RETURN((internal::OrExpr<internal::IsPointer<T>, internal::IsGenericValue<T> >), (GenericValue&))
    # 向 JSON 对象中添加成员，传入成员名、值和分配器
    AddMember(GenericValue& name, T value, Allocator& allocator) {
        # 创建一个 GenericValue 对象 v，用传入的值初始化
        GenericValue v(value);
        # 调用重载的 AddMember 函数，传入成员名、值和分配器，返回结果
        return AddMember(name, v, allocator);
    }
#if RAPIDJSON_HAS_CXX11_RVALUE_REFS
    // 如果支持 C++11 的右值引用，则定义 AddMember 方法，接受右值引用的 name 和 value
    GenericValue& AddMember(GenericValue&& name, GenericValue&& value, Allocator& allocator) {
        // 调用 AddMember 方法，传入 name 和 value 的右值引用
        return AddMember(name, value, allocator);
    }
    // 如果支持 C++11 的右值引用，则定义 AddMember 方法，接受右值引用的 name 和 左值引用的 value
    GenericValue& AddMember(GenericValue&& name, GenericValue& value, Allocator& allocator) {
        // 调用 AddMember 方法，传入 name 的右值引用和 value 的左值引用
        return AddMember(name, value, allocator);
    }
    // 如果支持 C++11 的右值引用，则定义 AddMember 方法，接受左值引用的 name 和 右值引用的 value
    GenericValue& AddMember(GenericValue& name, GenericValue&& value, Allocator& allocator) {
        // 调用 AddMember 方法，传入 name 的左值引用和 value 的右值引用
        return AddMember(name, value, allocator);
    }
    // 如果支持 C++11 的右值引用，则定义 AddMember 方法，接受常量字符串引用的 name 和 右值引用的 value
    GenericValue& AddMember(StringRefType name, GenericValue&& value, Allocator& allocator) {
        // 创建一个 GenericValue 对象 n，传入 name
        GenericValue n(name);
        // 调用 AddMember 方法，传入 n 和 value 的右值引用
        return AddMember(n, value, allocator);
    }
#endif // RAPIDJSON_HAS_CXX11_RVALUE_REFS

    //! 向对象添加成员（名称-值对）。
    /*! \param name 成员的名称，作为常量字符串引用。
        \param value 任何类型的值。
        \param allocator 用于重新分配内存的分配器。必须与之前使用的相同。通常使用 GenericDocument::GetAllocator()。
        \return 用于流畅 API 的值本身。
        \note 成功时，\c value 的所有权将转移到此对象。
        \pre  IsObject()
        \post value.IsNull()
        \note 分摊常数时间复杂度。
    */
    // 向对象添加成员（名称-值对）。
    GenericValue& AddMember(StringRefType name, GenericValue& value, Allocator& allocator) {
        // 创建一个 GenericValue 对象 n，传入 name
        GenericValue n(name);
        // 调用 AddMember 方法，传入 n 和 value 的左值引用
        return AddMember(n, value, allocator);
    }

    //! 向对象添加常量字符串值作为成员（名称-值对）。
    /*! \param name A constant string reference as name of member.
        \param value constant string reference as value of member.
        \param allocator    Allocator for reallocating memory. It must be the same one as used before. Commonly use GenericDocument::GetAllocator().
        \return The value itself for fluent API.
        \pre  IsObject()
        \note This overload is needed to avoid clashes with the generic primitive type AddMember(StringRefType,T,Allocator&) overload below.
        \note Amortized Constant time complexity.
    */
    # 添加一个成员（名称-值对）到对象中，名称和值都是常量字符串引用
    # allocator 用于重新分配内存，必须与之前使用的相同。通常使用 GenericDocument::GetAllocator()。
    # 返回值本身，用于流畅的 API。
    # 前提条件是对象必须是 IsObject()。
    # 注意：这个重载是为了避免与下面的通用原始类型 AddMember(StringRefType,T,Allocator&) 重载发生冲突。
    # 时间复杂度为摊销的常数时间。

    GenericValue& AddMember(StringRefType name, StringRefType value, Allocator& allocator) {
        GenericValue v(value);
        return AddMember(name, v, allocator);
    }

    //! Add any primitive value as member (name-value pair) to the object.
    /*! \tparam T Either \ref Type, \c int, \c unsigned, \c int64_t, \c uint64_t
        \param name A constant string reference as name of member.
        \param value Value of primitive type \c T as value of member
        \param allocator Allocator for reallocating memory. Commonly use GenericDocument::GetAllocator().
        \return The value itself for fluent API.
        \pre  IsObject()

        \note The source type \c T explicitly disallows all pointer types,
            especially (\c const) \ref Ch*.  This helps avoiding implicitly
            referencing character strings with insufficient lifetime, use
            \ref AddMember(StringRefType, GenericValue&, Allocator&) or \ref
            AddMember(StringRefType, StringRefType, Allocator&).
            All other pointer types would implicitly convert to \c bool,
            use an explicit cast instead, if needed.
        \note Amortized Constant time complexity.
    */
    # 将任何原始值作为成员（名称-值对）添加到对象中。
    # \tparam T 可以是 \ref Type, \c int, \c unsigned, \c int64_t, \c uint64_t 中的任何一种
    # \param name 作为成员名称的常量字符串引用
    # \param value 作为成员值的原始类型 \c T 的值
    # \param allocator 用于重新分配内存。通常使用 GenericDocument::GetAllocator()。
    # \return 返回值本身，用于流畅的 API。
    # \pre  对象必须是 IsObject()。
    # 注意：源类型 \c T 明确禁止所有指针类型，特别是 (\c const) \ref Ch*。这有助于避免隐式引用具有不足寿命的字符字符串，使用 \ref AddMember(StringRefType, GenericValue&, Allocator&) 或 \ref AddMember(StringRefType, StringRefType, Allocator&)。
    # 所有其他指针类型将隐式转换为 \c bool，如果需要，使用显式转换。
    # 时间复杂度为摊销的常数时间。

    template <typename T>
    RAPIDJSON_DISABLEIF_RETURN((internal::OrExpr<internal::IsPointer<T>, internal::IsGenericValue<T> >), (GenericValue&))
    # 根据给定的名称和值，向对象中添加一个成员
    AddMember(StringRefType name, T value, Allocator& allocator) {
        # 创建一个名为 n 的 GenericValue 对象，用给定的名称初始化
        GenericValue n(name);
        # 调用重载的 AddMember 函数，将名为 n 的成员和给定的值添加到对象中
        return AddMember(n, value, allocator);
    }

    # 移除对象中的所有成员
    /*! 此函数不会释放对象中的内存，即容量不会改变。
        \note 线性时间复杂度。
    */
    void RemoveAllMembers() {
        # 断言对象是一个对象类型
        RAPIDJSON_ASSERT(IsObject());
        # 调用 DoClearMembers 函数，清除对象中的所有成员
        DoClearMembers();
    }

    # 根据名称移除对象中的一个成员
    /*! \param name 要移除的成员的名称。
        \return 成员是否存在。
        \note 此函数可能重新排列对象成员的顺序。如果需要保留剩余成员的相对顺序，请使用 \ref
            EraseMember(ConstMemberIterator)。
        \note 线性时间复杂度。
    */
    bool RemoveMember(const Ch* name) {
        # 创建一个名为 n 的 GenericValue 对象，用给定的名称初始化
        GenericValue n(StringRef(name));
        # 调用重载的 RemoveMember 函数，根据名为 n 的成员名称移除对象中的一个成员
        return RemoveMember(n);
    }
#if RAPIDJSON_HAS_STDSTRING
    // 如果编译器支持 std::string，则使用 std::string 类型的参数来调用 RemoveMember 函数
    bool RemoveMember(const std::basic_string<Ch>& name) { return RemoveMember(GenericValue(StringRef(name))); }
#endif

    template <typename SourceAllocator>
    // 通过名称删除对象成员
    bool RemoveMember(const GenericValue<Encoding, SourceAllocator>& name) {
        // 查找指定名称的成员
        MemberIterator m = FindMember(name);
        // 如果找到了指定名称的成员
        if (m != MemberEnd()) {
            // 删除指定名称的成员
            RemoveMember(m);
            return true;
        }
        else
            return false;
    }

    //! 通过迭代器删除对象中的成员
    /*! \param m 成员迭代器（通过 FindMember() 或 MemberBegin() 获得）。
        \return 删除后的新迭代器。
        \note 此函数可能重新排列对象成员。如果需要保留剩余成员的相对顺序，请使用 \ref EraseMember(ConstMemberIterator)。
        \note 常数时间复杂度。
    */
    MemberIterator RemoveMember(MemberIterator m) {
        RAPIDJSON_ASSERT(IsObject());
        RAPIDJSON_ASSERT(data_.o.size > 0);
        RAPIDJSON_ASSERT(GetMembersPointer() != 0);
        RAPIDJSON_ASSERT(m >= MemberBegin() && m < MemberEnd());
        return DoRemoveMember(m);
    }

    //! 通过迭代器从对象中删除成员
    /*! \param pos 要删除的成员的迭代器
        \pre IsObject() == true && \ref MemberBegin() <= \c pos < \ref MemberEnd()
        \return 删除后的迭代器。
            如果迭代器 \c pos 指向最后一个元素，则返回 \ref MemberEnd() 迭代器。
        \note 此函数保留剩余对象成员的相对顺序。如果不需要这样做，请使用更高效的 \ref RemoveMember(MemberIterator)。
        \note 线性时间复杂度。
    */
    MemberIterator EraseMember(ConstMemberIterator pos) {
        return EraseMember(pos, pos +1);
    }

    //! 从对象中删除范围 [first, last) 内的成员
    /*! \param first iterator to the first member to remove
        \param last  iterator following the last member to remove
        \pre IsObject() == true && \ref MemberBegin() <= \c first <= \c last <= \ref MemberEnd()
        \return Iterator following the last removed element.
        \note This function preserves the relative order of the remaining object
            members.
        \note Linear time complexity.
    */
    # 根据给定的迭代器范围删除对象中的成员
    MemberIterator EraseMember(ConstMemberIterator first, ConstMemberIterator last) {
        # 断言对象类型为对象
        RAPIDJSON_ASSERT(IsObject());
        # 断言对象中有成员
        RAPIDJSON_ASSERT(data_.o.size > 0);
        # 断言获取成员指针不为空
        RAPIDJSON_ASSERT(GetMembersPointer() != 0);
        # 断言第一个迭代器在成员起始位置之后
        RAPIDJSON_ASSERT(first >= MemberBegin());
        # 断言第一个迭代器在最后一个迭代器之前
        RAPIDJSON_ASSERT(first <= last);
        # 断言最后一个迭代器在成员结束位置之前
        RAPIDJSON_ASSERT(last <= MemberEnd());
        # 调用私有函数执行删除操作
        return DoEraseMembers(first, last);
    }

    //! Erase a member in object by its name.
    /*! \param name Name of member to be removed.
        \return Whether the member existed.
        \note Linear time complexity.
    */
    # 根据成员名称删除对象中的成员
    bool EraseMember(const Ch* name) {
        # 创建包含名称的 GenericValue 对象
        GenericValue n(StringRef(name));
        # 调用重载的 EraseMember 函数执行删除操作
        return EraseMember(n);
    }
#if RAPIDJSON_HAS_STDSTRING
    // 如果编译器支持 std::string，则使用 std::string 类型的参数来删除成员
    bool EraseMember(const std::basic_string<Ch>& name) { return EraseMember(GenericValue(StringRef(name))); }
#endif

    // 使用模板函数，根据给定的名称删除成员
    template <typename SourceAllocator>
    bool EraseMember(const GenericValue<Encoding, SourceAllocator>& name) {
        // 查找给定名称的成员
        MemberIterator m = FindMember(name);
        // 如果找到了成员，则删除它并返回 true
        if (m != MemberEnd()) {
            EraseMember(m);
            return true;
        }
        // 如果未找到成员，则返回 false
        else
            return false;
    }

    // 返回当前值的对象类型
    Object GetObject() { RAPIDJSON_ASSERT(IsObject()); return Object(*this); }
    // 返回当前值的对象类型
    Object GetObj() { RAPIDJSON_ASSERT(IsObject()); return Object(*this); }
    // 返回当前值的对象类型（常量版本）
    ConstObject GetObject() const { RAPIDJSON_ASSERT(IsObject()); return ConstObject(*this); }
    // 返回当前值的对象类型（常量版本）
    ConstObject GetObj() const { RAPIDJSON_ASSERT(IsObject()); return ConstObject(*this); }

    //@}

    //!@name Array
    //@{

    //! 将当前值设置为空数组
    /*! \post IsArray == true */
    GenericValue& SetArray() { this->~GenericValue(); new (this) GenericValue(kArrayType); return *this; }

    //! 获取数组中元素的数量
    SizeType Size() const { RAPIDJSON_ASSERT(IsArray()); return data_.a.size; }

    //! 获取数组的容量
    SizeType Capacity() const { RAPIDJSON_ASSERT(IsArray()); return data_.a.capacity; }

    //! 检查数组是否为空
    bool Empty() const { RAPIDJSON_ASSERT(IsArray()); return data_.a.size == 0; }

    //! 清空数组中的所有元素
    /*! 此函数不会释放数组中的内存，即容量不会改变。
        \note 线性时间复杂度。
    */
    void Clear() {
        RAPIDJSON_ASSERT(IsArray());
        GenericValue* e = GetElementsPointer();
        for (GenericValue* v = e; v != e + data_.a.size; ++v)
            v->~GenericValue();
        data_.a.size = 0;
    }

    //! 通过索引从数组中获取元素
    /*! \pre IsArray() == true
        \param index 元素的从零开始的索引。
        \see operator[](T*)
    */
    # 重载操作符[]，用于访问数组中的元素
    GenericValue& operator[](SizeType index) {
        # 断言当前值为数组类型
        RAPIDJSON_ASSERT(IsArray());
        # 断言索引小于数组大小
        RAPIDJSON_ASSERT(index < data_.a.size);
        # 返回指向元素的指针
        return GetElementsPointer()[index];
    }
    # 重载操作符[]，用于访问数组中的元素（常量版本）
    const GenericValue& operator[](SizeType index) const { return const_cast<GenericValue&>(*this)[index]; }

    # 返回数组的起始迭代器
    # 先决条件：值为数组类型
    ValueIterator Begin() { RAPIDJSON_ASSERT(IsArray()); return GetElementsPointer(); }
    # 返回数组的结束迭代器
    # 先决条件：值为数组类型
    ValueIterator End() { RAPIDJSON_ASSERT(IsArray()); return GetElementsPointer() + data_.a.size; }
    # 返回常量数组的起始迭代器
    # 先决条件：值为数组类型
    ConstValueIterator Begin() const { return const_cast<GenericValue&>(*this).Begin(); }
    # 返回常量数组的结束迭代器
    # 先决条件：值为数组类型
    ConstValueIterator End() const { return const_cast<GenericValue&>(*this).End(); }

    # 请求数组具有足够的容量来存储元素
    # 参数 newCapacity：数组至少需要具有的容量
    # 参数 allocator：用于重新分配内存的分配器，必须与之前使用的相同
    # 返回值：为了流畅的 API 返回值本身
    # 注意：线性时间复杂度
    GenericValue& Reserve(SizeType newCapacity, Allocator &allocator) {
        # 断言当前值为数组类型
        RAPIDJSON_ASSERT(IsArray());
        # 如果新容量大于当前容量，则重新分配内存
        if (newCapacity > data_.a.capacity) {
            SetElementsPointer(reinterpret_cast<GenericValue*>(allocator.Realloc(GetElementsPointer(), data_.a.capacity * sizeof(GenericValue), newCapacity * sizeof(GenericValue))));
            data_.a.capacity = newCapacity;
        }
        return *this;
    }

    # 在数组末尾追加一个 GenericValue
    /*! \param value        Value to be appended.  // 参数 value：要追加的值
        \param allocator    Allocator for reallocating memory. It must be the same one as used before. Commonly use GenericDocument::GetAllocator().  // 参数 allocator：用于重新分配内存的分配器。必须与之前使用的相同。通常使用 GenericDocument::GetAllocator()。
        \pre IsArray() == true  // 先决条件：IsArray() == true
        \post value.IsNull() == true  // 后置条件：value.IsNull() == true
        \return The value itself for fluent API.  // 返回值：为流畅 API 返回值本身
        \note The ownership of \c value will be transferred to this array on success.  // 注意：成功后，\c value 的所有权将转移到此数组
        \note If the number of elements to be appended is known, calls Reserve() once first may be more efficient.  // 注意：如果知道要追加的元素数量，先调用 Reserve() 一次可能更有效率。
        \note Amortized constant time complexity.  // 注意：摊销常数时间复杂度。
    */
    GenericValue& PushBack(GenericValue& value, Allocator& allocator) {
        RAPIDJSON_ASSERT(IsArray());  // 断言：IsArray() == true
        if (data_.a.size >= data_.a.capacity)  // 如果当前数组大小大于等于容量
            Reserve(data_.a.capacity == 0 ? kDefaultArrayCapacity : (data_.a.capacity + (data_.a.capacity + 1) / 2), allocator);  // 调整数组容量
        GetElementsPointer()[data_.a.size++].RawAssign(value);  // 获取元素指针，将 value 赋值给数组最后一个元素
        return *this;  // 返回值本身，用于流畅 API
    }
#if RAPIDJSON_HAS_CXX11_RVALUE_REFS
    // 如果支持 C++11 的右值引用，则定义 PushBack 函数，接受右值引用参数
    GenericValue& PushBack(GenericValue&& value, Allocator& allocator) {
        return PushBack(value, allocator);
    }
#endif // RAPIDJSON_HAS_CXX11_RVALUE_REFS

    //! 在数组末尾追加一个常量字符串引用
    /*! \param value        要追加的常量字符串引用
        \param allocator    用于重新分配内存的分配器。必须与之前使用的相同。通常使用 GenericDocument::GetAllocator()。
        \pre IsArray() == true
        \return 用于流畅 API 的值本身。
        \note 如果要追加的元素数量已知，则首先调用 Reserve() 一次可能更有效。
        \note 分摊常数时间复杂度。
        \see GenericStringRef
    */
    GenericValue& PushBack(StringRefType value, Allocator& allocator) {
        return (*this).template PushBack<StringRefType>(value, allocator);
    }

    //! 在数组末尾追加一个基本值
    /*! \tparam T 要么是 \ref Type，\c int，\c unsigned，\c int64_t，\c uint64_t
        \param value 要追加的基本类型 T 的值。
        \param allocator 用于重新分配内存的分配器。必须与之前使用的相同。通常使用 GenericDocument::GetAllocator()。
        \pre IsArray() == true
        \return 用于流畅 API 的值本身。
        \note 如果要追加的元素数量已知，则首先调用 Reserve() 一次可能更有效。

        \note 源类型 \c T 明确禁止所有指针类型，
            特别是 (\c const) \ref Ch*。这有助于避免隐式引用具有不足寿命的字符字符串，使用
            \ref PushBack(GenericValue&, Allocator&) 或 \ref
            PushBack(StringRefType, Allocator&)。
            所有其他指针类型将隐式转换为 \c bool，
            如有需要，使用显式转换。
        \note 分摊常数时间复杂度。
    */
    // 如果 T 不是指针类型或者 GenericValue 类型，则返回 GenericValue 的引用
    template <typename T>
    RAPIDJSON_DISABLEIF_RETURN((internal::OrExpr<internal::IsPointer<T>, internal::IsGenericValue<T> >), (GenericValue&))
    PushBack(T value, Allocator& allocator) {
        // 创建一个 GenericValue 对象 v，并用 value 初始化
        GenericValue v(value);
        // 调用 PushBack 方法将 v 添加到数组中，并返回结果
        return PushBack(v, allocator);
    }

    //! 移除数组中的最后一个元素
    /*!
        \note 时间复杂度为常数。
    */
    GenericValue& PopBack() {
        // 断言当前对象是数组类型
        RAPIDJSON_ASSERT(IsArray());
        // 断言数组不为空
        RAPIDJSON_ASSERT(!Empty());
        // 获取数组元素的指针，移除最后一个元素，并调用其析构函数
        GetElementsPointer()[--data_.a.size].~GenericValue();
        // 返回当前对象的引用
        return *this;
    }

    //! 通过迭代器移除数组中的一个元素
    /*!
        \param pos 要移除的元素的迭代器
        \pre IsArray() == true && \ref Begin() <= \c pos < \ref End()
        \return 移除元素后的迭代器。如果迭代器 pos 指向最后一个元素，则返回 End() 迭代器。
        \note 时间复杂度为线性。
    */
    ValueIterator Erase(ConstValueIterator pos) {
        // 调用 Erase 方法移除 pos 指向的元素
        return Erase(pos, pos + 1);
    }

    //! 移除数组中范围 [first, last) 内的元素
    /*!
        \param first 要移除的第一个元素的迭代器
        \param last  要移除的最后一个元素的后一个迭代器
        \pre IsArray() == true && \ref Begin() <= \c first <= \c last <= \ref End()
        \return 移除元素后的最后一个元素的迭代器。
        \note 时间复杂度为线性。
    */
    // 从数组中删除指定范围的元素，并返回指向被删除元素后面的第一个元素的迭代器
    ValueIterator Erase(ConstValueIterator first, ConstValueIterator last) {
        // 断言当前值为数组类型
        RAPIDJSON_ASSERT(IsArray());
        // 断言数组大小大于0
        RAPIDJSON_ASSERT(data_.a.size > 0);
        // 断言获取元素指针不为空
        RAPIDJSON_ASSERT(GetElementsPointer() != 0);
        // 断言first迭代器在Begin()和End()之间
        RAPIDJSON_ASSERT(first >= Begin());
        // 断言last迭代器在first和End()之间
        RAPIDJSON_ASSERT(first <= last);
        // 断言last迭代器在first和End()之间
        RAPIDJSON_ASSERT(last <= End());
        // 计算被删除元素后面的第一个元素的迭代器
        ValueIterator pos = Begin() + (first - Begin());
        // 逐个调用被删除元素的析构函数
        for (ValueIterator itr = pos; itr != last; ++itr)
            itr->~GenericValue();
        // 移动剩余元素到被删除元素的位置
        std::memmove(static_cast<void*>(pos), last, static_cast<size_t>(End() - last) * sizeof(GenericValue));
        // 更新数组大小
        data_.a.size -= static_cast<SizeType>(last - first);
        // 返回指向被删除元素后面的第一个元素的迭代器
        return pos;
    }

    // 获取当前值的数组类型
    Array GetArray() { RAPIDJSON_ASSERT(IsArray()); return Array(*this); }
    // 获取当前值的数组类型（常量版本）
    ConstArray GetArray() const { RAPIDJSON_ASSERT(IsArray()); return ConstArray(*this); }

    //@}

    //!@name Number
    //@{

    // 获取当前值的int类型
    int GetInt() const          { RAPIDJSON_ASSERT(data_.f.flags & kIntFlag);   return data_.n.i.i;   }
    // 获取当前值的unsigned int类型
    unsigned GetUint() const    { RAPIDJSON_ASSERT(data_.f.flags & kUintFlag);  return data_.n.u.u;   }
    // 获取当前值的int64_t类型
    int64_t GetInt64() const    { RAPIDJSON_ASSERT(data_.f.flags & kInt64Flag); return data_.n.i64; }
    // 获取当前值的uint64_t类型
    uint64_t GetUint64() const  { RAPIDJSON_ASSERT(data_.f.flags & kUint64Flag); return data_.n.u64; }

    //! 获取当前值的double类型
    /*! \note 如果值为64位整数类型，可能会失去精度。使用 \c IsLosslessDouble() 检查转换是否无损。
    */
    // 返回值为 double 类型的函数，断言当前值为数字类型
    double GetDouble() const {
        RAPIDJSON_ASSERT(IsNumber());
        // 如果当前值的标志位包含 kDoubleFlag，则返回精确类型的 double 值，无需转换
        if ((data_.f.flags & kDoubleFlag) != 0)                return data_.n.d;   // exact type, no conversion.
        // 如果当前值的标志位包含 kIntFlag，则将 int 转换为 double 后返回
        if ((data_.f.flags & kIntFlag) != 0)                   return data_.n.i.i; // int -> double
        // 如果当前值的标志位包含 kUintFlag，则将 unsigned 转换为 double 后返回
        if ((data_.f.flags & kUintFlag) != 0)                  return data_.n.u.u; // unsigned -> double
        // 如果当前值的标志位包含 kInt64Flag，则将 int64_t 转换为 double 后返回（可能会丢失精度）
        if ((data_.f.flags & kInt64Flag) != 0)                 return static_cast<double>(data_.n.i64); // int64_t -> double (may lose precision)
        // 断言当前值的标志位包含 kUint64Flag，将 uint64_t 转换为 double 后返回（可能会丢失精度）
        RAPIDJSON_ASSERT((data_.f.flags & kUint64Flag) != 0);  return static_cast<double>(data_.n.u64); // uint64_t -> double (may lose precision)
    }

    //! 返回值为 float 类型的函数，将 GetDouble() 返回的 double 值转换为 float 后返回
    /*! \note 如果值为 64 位整数类型，可能会丢失精度。使用 \c IsLosslessFloat() 检查转换是否无损失。
    */
    float GetFloat() const {
        return static_cast<float>(GetDouble());
    }

    // 设置当前值为 int 类型，并返回当前对象的引用
    GenericValue& SetInt(int i)             { this->~GenericValue(); new (this) GenericValue(i);    return *this; }
    // 设置当前值为 unsigned 类型，并返回当前对象的引用
    GenericValue& SetUint(unsigned u)       { this->~GenericValue(); new (this) GenericValue(u);    return *this; }
    // 设置当前值为 int64_t 类型，并返回当前对象的引用
    GenericValue& SetInt64(int64_t i64)     { this->~GenericValue(); new (this) GenericValue(i64);  return *this; }
    // 设置当前值为 uint64_t 类型，并返回当前对象的引用
    GenericValue& SetUint64(uint64_t u64)   { this->~GenericValue(); new (this) GenericValue(u64);  return *this; }
    // 设置当前值为 double 类型，并返回当前对象的引用
    GenericValue& SetDouble(double d)       { this->~GenericValue(); new (this) GenericValue(d);    return *this; }
    // 设置当前值为 float 类型，并返回当前对象的引用
    GenericValue& SetFloat(float f)         { this->~GenericValue(); new (this) GenericValue(static_cast<double>(f)); return *this; }

    //@}

    //!@name String
    //@{

    // 返回当前值的字符串表示，断言当前值为字符串类型
    const Ch* GetString() const { RAPIDJSON_ASSERT(IsString()); return DataString(data_); }

    // 返回字符串的长度
    /*! 由于 rapidjson 允许在 json 字符串中使用 "\\u0000"，因此 strlen(v.GetString()) 可能不等于 v.GetStringLength()。
    */
    # 获取字符串的长度，如果不是字符串类型则断言失败
    SizeType GetStringLength() const { RAPIDJSON_ASSERT(IsString()); return DataStringLength(data_); }

    # 设置该值为一个字符串，不复制源字符串
    # 这个版本在提供长度的情况下性能更好，并且支持包含空字符的字符串
    # s：源字符串指针
    # length：源字符串的长度，不包括结尾的空字符
    # 返回：值本身，用于流畅的 API
    # 设置后：IsString() == true && GetString() == s && GetStringLength() == length
    # 参见：SetString(StringRefType)
    GenericValue& SetString(const Ch* s, SizeType length) { return SetString(StringRef(s, length)); }

    # 设置该值为一个字符串，不复制源字符串
    # s：源字符串引用
    # 返回：值本身，用于流畅的 API
    # 设置后：IsString() == true && GetString() == s && GetStringLength() == s.length
    GenericValue& SetString(StringRefType s) { this->~GenericValue(); SetStringRaw(s); return *this; }

    # 通过从源字符串复制来设置该值为一个字符串
    # 这个版本在提供长度的情况下性能更好，并且支持包含空字符的字符串
    # s：源字符串
    # length：源字符串的长度，不包括结尾的空字符
    # allocator：用于分配复制缓冲区的分配器。通常使用 GenericDocument::GetAllocator()
    # 返回：值本身，用于流畅的 API
    # 设置后：IsString() == true && GetString() != s && strcmp(GetString(),s) == 0 && GetStringLength() == length
    GenericValue& SetString(const Ch* s, SizeType length, Allocator& allocator) { return SetString(StringRef(s, length), allocator); }

    # 通过从源字符串复制来设置该值为一个字符串
    # 用于设置值为字符串，从源字符串复制
    # 参数 s：源字符串
    # 参数 allocator：用于分配复制缓冲区的分配器。通常使用 GenericDocument::GetAllocator()
    # 返回值：为了流畅的 API，返回值本身
    # 设置值为字符串，从源字符串复制
    # 参数 s：源字符串引用
    # 参数 allocator：用于分配复制缓冲区的分配器。通常使用 GenericDocument::GetAllocator()
    # 返回值：为了流畅的 API，返回值本身
#if RAPIDJSON_HAS_STDSTRING
    //! Set this value as a string by copying from source string.
    /*! \param s source string.
        \param allocator Allocator for allocating copied buffer. Commonly use GenericDocument::GetAllocator().
        \return The value itself for fluent API.
        \post IsString() == true && GetString() != s.data() && strcmp(GetString(),s.data() == 0 && GetStringLength() == s.size()
        \note Requires the definition of the preprocessor symbol \ref RAPIDJSON_HAS_STDSTRING.
    */
    GenericValue& SetString(const std::basic_string<Ch>& s, Allocator& allocator) { return SetString(StringRef(s), allocator); }
#endif

    //@}

    //!@name Array
    //@{

    //! Templated version for checking whether this value is type T.
    /*!
        \tparam T Either \c bool, \c int, \c unsigned, \c int64_t, \c uint64_t, \c double, \c float, \c const \c char*, \c std::basic_string<Ch>
    */
    template <typename T>
    bool Is() const { return internal::TypeHelper<ValueType, T>::Is(*this); }

    template <typename T>
    T Get() const { return internal::TypeHelper<ValueType, T>::Get(*this); }

    template <typename T>
    T Get() { return internal::TypeHelper<ValueType, T>::Get(*this); }

    template<typename T>
    ValueType& Set(const T& data) { return internal::TypeHelper<ValueType, T>::Set(*this, data); }

    template<typename T>
    ValueType& Set(const T& data, AllocatorType& allocator) { return internal::TypeHelper<ValueType, T>::Set(*this, data, allocator); }

    //@}

    //! Generate events of this value to a Handler.
    /*! This function adopts the GoF visitor pattern.
        Typical usage is to output this JSON value as JSON text via Writer, which is a Handler.
        It can also be used to deep clone this value via GenericDocument, which is also a Handler.
        \tparam Handler type of handler.
        \param handler An object implementing concept Handler.
    */
    template <typename Handler>
    // 判断当前值的类型，并根据不同类型执行不同的操作
    bool Accept(Handler& handler) const {
        // 根据值的类型进行不同的处理
        switch(GetType()) {
        // 如果是空值类型，则调用处理程序的 Null() 方法
        case kNullType:     return handler.Null();
        // 如果是布尔假类型，则调用处理程序的 Bool(false) 方法
        case kFalseType:    return handler.Bool(false);
        // 如果是布尔真类型，则调用处理程序的 Bool(true) 方法
        case kTrueType:     return handler.Bool(true);

        // 如果是对象类型
        case kObjectType:
            // 如果处理程序的 StartObject() 方法返回 false，则返回 false
            if (RAPIDJSON_UNLIKELY(!handler.StartObject()))
                return false;
            // 遍历对象的成员，依次处理每个成员
            for (ConstMemberIterator m = MemberBegin(); m != MemberEnd(); ++m) {
                // 断言成员的名称是字符串类型
                RAPIDJSON_ASSERT(m->name.IsString()); // User may change the type of name by MemberIterator.
                // 如果处理程序的 Key() 方法返回 false，则返回 false
                if (RAPIDJSON_UNLIKELY(!handler.Key(m->name.GetString(), m->name.GetStringLength(), (m->name.data_.f.flags & kCopyFlag) != 0)))
                    return false;
                // 如果成员的值的 Accept() 方法返回 false，则返回 false
                if (RAPIDJSON_UNLIKELY(!m->value.Accept(handler)))
                    return false;
            }
            // 调用处理程序的 EndObject() 方法，并传入对象的大小，返回结果
            return handler.EndObject(data_.o.size);

        // 如果是数组类型
        case kArrayType:
            // 如果处理程序的 StartArray() 方法返回 false，则返回 false
            if (RAPIDJSON_UNLIKELY(!handler.StartArray()))
                return false;
            // 遍历数组的元素，依次处理每个元素
            for (ConstValueIterator v = Begin(); v != End(); ++v)
                // 如果元素的 Accept() 方法返回 false，则返回 false
                if (RAPIDJSON_UNLIKELY(!v->Accept(handler)))
                    return false;
            // 调用处理程序的 EndArray() 方法，并传入数组的大小，返回结果
            return handler.EndArray(data_.a.size);

        // 如果是字符串类型，则调用处理程序的 String() 方法
        case kStringType:
            return handler.String(GetString(), GetStringLength(), (data_.f.flags & kCopyFlag) != 0);

        // 如果是其他类型（数字类型）
        default:
            // 断言值的类型是数字类型
            RAPIDJSON_ASSERT(GetType() == kNumberType);
            // 根据具体的数字类型调用处理程序对应的方法
            if (IsDouble())         return handler.Double(data_.n.d);
            else if (IsInt())       return handler.Int(data_.n.i.i);
            else if (IsUint())      return handler.Uint(data_.n.u.u);
            else if (IsInt64())     return handler.Int64(data_.n.i64);
            else                    return handler.Uint64(data_.n.u64);
        }
    }
private:
    // 声明 GenericValue 类和 GenericDocument 类为友元类
    template <typename, typename> friend class GenericValue;
    template <typename, typename, typename> friend class GenericDocument;

    // 定义枚举值，表示不同类型的标志位
    enum {
        kBoolFlag       = 0x0008,  // 布尔类型标志位
        kNumberFlag     = 0x0010,  // 数值类型标志位
        kIntFlag        = 0x0020,  // 整数类型标志位
        kUintFlag       = 0x0040,  // 无符号整数类型标志位
        kInt64Flag      = 0x0080,  // 64位整数类型标志位
        kUint64Flag     = 0x0100,  // 64位无符号整数类型标志位
        kDoubleFlag     = 0x0200,  // 双精度浮点数类型标志位
        kStringFlag     = 0x0400,  // 字符串类型标志位
        kCopyFlag       = 0x0800,  // 复制标志位
        kInlineStrFlag  = 0x1000,  // 内联字符串标志位

        // 不同类型的初始标志位
        kNullFlag = kNullType,  // 空类型标志位
        kTrueFlag = static_cast<int>(kTrueType) | static_cast<int>(kBoolFlag),  // 真值类型标志位
        kFalseFlag = static_cast<int>(kFalseType) | static_cast<int>(kBoolFlag),  // 假值类型标志位
        kNumberIntFlag = static_cast<int>(kNumberType) | static_cast<int>(kNumberFlag | kIntFlag | kInt64Flag),  // 整数类型标志位
        kNumberUintFlag = static_cast<int>(kNumberType) | static_cast<int>(kNumberFlag | kUintFlag | kUint64Flag | kInt64Flag),  // 无符号整数类型标志位
        kNumberInt64Flag = static_cast<int>(kNumberType) | static_cast<int>(kNumberFlag | kInt64Flag),  // 64位整数类型标志位
        kNumberUint64Flag = static_cast<int>(kNumberType) | static_cast<int>(kNumberFlag | kUint64Flag),  // 64位无符号整数类型标志位
        kNumberDoubleFlag = static_cast<int>(kNumberType) | static_cast<int>(kNumberFlag | kDoubleFlag),  // 双精度浮点数类型标志位
        kNumberAnyFlag = static_cast<int>(kNumberType) | static_cast<int>(kNumberFlag | kIntFlag | kInt64Flag | kUintFlag | kUint64Flag | kDoubleFlag),  // 任意数值类型标志位
        kConstStringFlag = static_cast<int>(kStringType) | static_cast<int>(kStringFlag),  // 常量字符串类型标志位
        kCopyStringFlag = static_cast<int>(kStringType) | static_cast<int>(kStringFlag | kCopyFlag),  // 复制字符串类型标志位
        kShortStringFlag = static_cast<int>(kStringType) | static_cast<int>(kStringFlag | kCopyFlag | kInlineStrFlag),  // 短字符串类型标志位
        kObjectFlag = kObjectType,  // 对象类型标志位
        kArrayFlag = kArrayType,  // 数组类型标志位

        kTypeMask = 0x07  // 类型掩码
    };
    # 定义静态常量，表示默认的数组容量
    static const SizeType kDefaultArrayCapacity = RAPIDJSON_VALUE_DEFAULT_ARRAY_CAPACITY;
    # 定义静态常量，表示默认的对象容量
    static const SizeType kDefaultObjectCapacity = RAPIDJSON_VALUE_DEFAULT_OBJECT_CAPACITY;

    # 定义结构体 Flag
    struct Flag {
#if RAPIDJSON_48BITPOINTER_OPTIMIZATION
        // 如果启用了48位指针优化，则分配足够空间来存储2个SizeType和一个48位指针
        char payload[sizeof(SizeType) * 2 + 6];     // 2 x SizeType + lower 48-bit pointer
#elif RAPIDJSON_64BIT
        // 如果是64位系统，则分配足够空间来存储2个SizeType、一个void指针和6个填充字节
        char payload[sizeof(SizeType) * 2 + sizeof(void*) + 6]; // 6 padding bytes
#else
        // 如果是32位系统，则分配足够空间来存储2个SizeType、一个void指针和2个填充字节
        char payload[sizeof(SizeType) * 2 + sizeof(void*) + 2]; // 2 padding bytes
#endif
        // 16位整数类型，用于存储标志
        uint16_t flags;
    };

    // 字符串结构体，包含长度、哈希码和指向字符的指针
    struct String {
        SizeType length;  // 字符串长度
        SizeType hashcode;  //!< reserved  // 哈希码，保留字段
        const Ch* str;  // 字符串指针
    };  // 12 bytes in 32-bit mode, 16 bytes in 64-bit mode

    // 实现细节：ShortString可以表示最多MaxSize个字符的以零结尾的字符串
    // 并且通过在最后一个字符str[LenPos]中存储一个值来确定包含字符串的长度
    // 如果要存储的字符串具有最大长度MaxSize，则str[LenPos]将为0，因此也充当字符串终结符
    // 从该值中获取字符串长度只需使用"MaxSize - str[LenPos]"
    // 这允许在32位模式下存储13个字符的字符串，在64位模式下存储21个字符的字符串
    // 对于RAPIDJSON_48BITPOINTER_OPTIMIZATION=1，可以内联存储13个字符的字符串（对于UTF8编码的字符串）
    struct ShortString {
        enum { MaxChars = sizeof(static_cast<Flag*>(0)->payload) / sizeof(Ch), MaxSize = MaxChars - 1, LenPos = MaxSize };
        Ch str[MaxChars];  // 字符数组

        // 判断给定长度的字符串是否可用
        inline static bool Usable(SizeType len) { return                       (MaxSize >= len); }
        // 设置字符串长度
        inline void     SetLength(SizeType len) { str[LenPos] = static_cast<Ch>(MaxSize -  len); }
        // 获取字符串长度
        inline SizeType GetLength() const       { return  static_cast<SizeType>(MaxSize -  str[LenPos]); }
    };  // at most as many bytes as "String" above => 12 bytes in 32-bit mode, 16 bytes in 64-bit mode

    // 通过使用适当的二进制布局，不需要进行不同整数类型的转换即可检索
    union Number {
#if RAPIDJSON_ENDIAN == RAPIDJSON_LITTLEENDIAN
    // 如果是小端序，定义结构体 I 包含整型和填充字节，结构体 U 包含无符号整型和填充字节
    struct I {
        int i;
        char padding[4];
    }i;
    struct U {
        unsigned u;
        char padding2[4];
    }u;
#else
    // 如果不是小端序，定义结构体 I 包含填充字节和整型，结构体 U 包含填充字节和无符号整型
    struct I {
        char padding[4];
        int i;
    }i;
    struct U {
        char padding2[4];
        unsigned u;
    }u;
#endif
    // 定义64位有符号整型、64位无符号整型和双精度浮点数
    int64_t i64;
    uint64_t u64;
    double d;
};  // 8 bytes

// 定义 ObjectData 结构体，包含 size、capacity 和 members
struct ObjectData {
    SizeType size;
    SizeType capacity;
    Member* members;
};  // 12 bytes in 32-bit mode, 16 bytes in 64-bit mode

// 定义 ArrayData 结构体，包含 size、capacity 和 elements
struct ArrayData {
    SizeType size;
    SizeType capacity;
    GenericValue* elements;
};  // 12 bytes in 32-bit mode, 16 bytes in 64-bit mode

// 定义联合体 Data，包含 String、ShortString、Number、ObjectData、ArrayData 和 Flag
union Data {
    String s;
    ShortString ss;
    Number n;
    ObjectData o;
    ArrayData a;
    Flag f;
};  // 16 bytes in 32-bit mode, 24 bytes in 64-bit mode, 16 bytes in 64-bit with RAPIDJSON_48BITPOINTER_OPTIMIZATION

// 定义 DataString 函数，根据 Data 返回字符串指针
static RAPIDJSON_FORCEINLINE const Ch* DataString(const Data& data) {
    return (data.f.flags & kInlineStrFlag) ? data.ss.str : RAPIDJSON_GETPOINTER(Ch, data.s.str);
}
// 定义 DataStringLength 函数，根据 Data 返回字符串长度
static RAPIDJSON_FORCEINLINE SizeType DataStringLength(const Data& data) {
    return (data.f.flags & kInlineStrFlag) ? data.ss.GetLength() : data.s.length;
}

// 定义 GetStringPointer 函数，返回字符串指针
RAPIDJSON_FORCEINLINE const Ch* GetStringPointer() const { return RAPIDJSON_GETPOINTER(Ch, data_.s.str); }
// 定义 SetStringPointer 函数，设置字符串指针
RAPIDJSON_FORCEINLINE const Ch* SetStringPointer(const Ch* str) { return RAPIDJSON_SETPOINTER(Ch, data_.s.str, str); }
// 定义 GetElementsPointer 函数，返回元素指针
RAPIDJSON_FORCEINLINE GenericValue* GetElementsPointer() const { return RAPIDJSON_GETPOINTER(GenericValue, data_.a.elements); }
// 定义 SetElementsPointer 函数，设置元素指针
RAPIDJSON_FORCEINLINE GenericValue* SetElementsPointer(GenericValue* elements) { return RAPIDJSON_SETPOINTER(GenericValue, data_.a.elements, elements); }
    # 返回成员指针的内联函数
    RAPIDJSON_FORCEINLINE Member* GetMembersPointer() const { return RAPIDJSON_GETPOINTER(Member, data_.o.members); }
    # 设置成员指针的内联函数
    RAPIDJSON_FORCEINLINE Member* SetMembersPointer(Member* members) { return RAPIDJSON_SETPOINTER(Member, data_.o.members, members); }
#if RAPIDJSON_USE_MEMBERSMAP
    // 如果定义了RAPIDJSON_USE_MEMBERSMAP，则执行以下代码块

    struct MapTraits {
        // 定义MapTraits结构体
        struct Less {
            // 定义Less结构体，用于比较Data类型的数据
            bool operator()(const Data& s1, const Data& s2) const {
                // 重载()运算符，比较两个Data类型的数据
                SizeType n1 = DataStringLength(s1), n2 = DataStringLength(s2);
                // 获取两个Data类型数据的长度
                int cmp = std::memcmp(DataString(s1), DataString(s2), sizeof(Ch) * (n1 < n2 ? n1 : n2));
                // 比较两个Data类型数据的内容
                return cmp < 0 || (cmp == 0 && n1 < n2);
                // 返回比较结果
            }
        };
        // 定义Less结构体

        typedef std::pair<const Data, SizeType> Pair;
        // 定义Pair类型，包含const Data和SizeType
        typedef std::multimap<Data, SizeType, Less, StdAllocator<Pair, Allocator> > Map;
        // 定义Map类型，使用multimap容器，键为Data，值为SizeType，使用Less进行比较，使用StdAllocator进行内存分配
        typedef typename Map::iterator Iterator;
        // 定义Iterator类型，为Map的迭代器
    };
    // 定义MapTraits结构体

    typedef typename MapTraits::Map         Map;
    // 定义Map类型，使用MapTraits中的Map
    typedef typename MapTraits::Less        MapLess;
    // 定义MapLess类型，使用MapTraits中的Less
    typedef typename MapTraits::Pair        MapPair;
    // 定义MapPair类型，使用MapTraits中的Pair
    typedef typename MapTraits::Iterator    MapIterator;
    // 定义MapIterator类型，使用MapTraits中的Iterator

    //
    // Layout of the members' map/array, re(al)located according to the needed capacity:
    //
    //    {Map*}<>{capacity}<>{Member[capacity]}<>{MapIterator[capacity]}
    //
    // (where <> stands for the RAPIDJSON_ALIGN-ment, if needed)
    //
    // 成员的映射/数组的布局，根据需要的容量重新分配：
    //    {Map*}<>{capacity}<>{Member[capacity]}<>{MapIterator[capacity]}
    // (其中<>表示RAPIDJSON_ALIGN-ment，如果需要的话)

    static RAPIDJSON_FORCEINLINE size_t GetMapLayoutSize(SizeType capacity) {
        // 定义GetMapLayoutSize函数，根据容量返回布局大小
        return RAPIDJSON_ALIGN(sizeof(Map*)) +
               RAPIDJSON_ALIGN(sizeof(SizeType)) +
               RAPIDJSON_ALIGN(capacity * sizeof(Member)) +
               capacity * sizeof(MapIterator);
        // 返回布局大小
    }

    static RAPIDJSON_FORCEINLINE SizeType &GetMapCapacity(Map* &map) {
        // 定义GetMapCapacity函数，返回Map的容量
        return *reinterpret_cast<SizeType*>(reinterpret_cast<uintptr_t>(&map) +
                                            RAPIDJSON_ALIGN(sizeof(Map*)));
        // 返回Map的容量
    }

    static RAPIDJSON_FORCEINLINE Member* GetMapMembers(Map* &map) {
        // 定义GetMapMembers函数，返回Map的成员
        return reinterpret_cast<Member*>(reinterpret_cast<uintptr_t>(&map) +
                                         RAPIDJSON_ALIGN(sizeof(Map*)) +
                                         RAPIDJSON_ALIGN(sizeof(SizeType)));
        // 返回Map的成员
    }
#endif
    // 获取 Map 对象的迭代器指针
    static RAPIDJSON_FORCEINLINE MapIterator* GetMapIterators(Map* &map) {
        // 通过指针运算获取迭代器指针
        return reinterpret_cast<MapIterator*>(reinterpret_cast<uintptr_t>(&map) +
                                              RAPIDJSON_ALIGN(sizeof(Map*)) +
                                              RAPIDJSON_ALIGN(sizeof(SizeType)) +
                                              RAPIDJSON_ALIGN(GetMapCapacity(map) * sizeof(Member)));
    }

    // 获取 Map 对象的引用
    static RAPIDJSON_FORCEINLINE Map* &GetMap(Member* members) {
        // 断言成员不为空
        RAPIDJSON_ASSERT(members != 0);
        // 通过指针运算获取 Map 对象的引用
        return *reinterpret_cast<Map**>(reinterpret_cast<uintptr_t>(members) -
                                        RAPIDJSON_ALIGN(sizeof(SizeType)) -
                                        RAPIDJSON_ALIGN(sizeof(Map*)));
    }

    // 一些编译器的调试机制要求销毁所有迭代器，以进行计算
    RAPIDJSON_FORCEINLINE MapIterator DropMapIterator(MapIterator& rhs) {
#if RAPIDJSON_HAS_CXX11
        // 如果支持 C++11，则使用移动语义构造 MapIterator 对象
        MapIterator ret = std::move(rhs);
#else
        // 否则使用拷贝构造 MapIterator 对象
        MapIterator ret = rhs;
#endif
        // 调用 rhs 对象的析构函数
        rhs.~MapIterator();
        // 返回构造好的 MapIterator 对象
        return ret;
    }

    // 重新分配 Map 对象的内存
    Map* &DoReallocMap(Map** oldMap, SizeType newCapacity, Allocator& allocator) {
        // 分配新的 Map 对象内存
        Map **newMap = static_cast<Map**>(allocator.Malloc(GetMapLayoutSize(newCapacity)));
        // 设置新 Map 对象的容量
        GetMapCapacity(*newMap) = newCapacity;
        // 如果旧 Map 对象为空
        if (!oldMap) {
            // 在新 Map 对象内存上构造一个新的 Map 对象
            *newMap = new (allocator.Malloc(sizeof(Map))) Map(MapLess(), allocator);
        }
        else {
            // 否则将旧 Map 对象的内容拷贝到新 Map 对象
            *newMap = *oldMap;
            size_t count = (*oldMap)->size();
            // 使用内存拷贝函数拷贝 Member 对象
            std::memcpy(static_cast<void*>(GetMapMembers(*newMap)),
                        static_cast<void*>(GetMapMembers(*oldMap)),
                        count * sizeof(Member));
            MapIterator *oldIt = GetMapIterators(*oldMap),
                        *newIt = GetMapIterators(*newMap);
            // 逐个构造新的 MapIterator 对象
            while (count--) {
                new (&newIt[count]) MapIterator(DropMapIterator(oldIt[count]));
            }
            // 释放旧 Map 对象的内存
            Allocator::Free(oldMap);
        }
        // 返回新 Map 对象的指针
        return *newMap;
    }

    // 分配 Map 对象的内存并返回 Member 对象指针
    RAPIDJSON_FORCEINLINE Member* DoAllocMembers(SizeType capacity, Allocator& allocator) {
        return GetMapMembers(DoReallocMap(0, capacity, allocator));
    }

    // 预留足够的内存给 Member 对象
    void DoReserveMembers(SizeType newCapacity, Allocator& allocator) {
        ObjectData& o = data_.o;
        // 如果需要的容量大于当前容量
        if (newCapacity > o.capacity) {
            Member* oldMembers = GetMembersPointer();
            Map **oldMap = oldMembers ? &GetMap(oldMembers) : 0,
                *&newMap = DoReallocMap(oldMap, newCapacity, allocator);
            // 设置新的 Member 对象指针
            RAPIDJSON_SETPOINTER(Member, o.members, GetMapMembers(newMap));
            // 更新容量
            o.capacity = newCapacity;
        }
    }

    template <typename SourceAllocator>
    # 查找指定成员，并返回成员迭代器
    MemberIterator DoFindMember(const GenericValue<Encoding, SourceAllocator>& name) {
        # 如果成员指针存在
        if (Member* members = GetMembersPointer()) {
            # 获取成员对应的映射
            Map* &map = GetMap(members);
            # 通过名称查找成员迭代器
            MapIterator mit = map->find(reinterpret_cast<const Data&>(name.data_));
            # 如果找到了对应的成员
            if (mit != map->end()) {
                # 返回成员迭代器
                return MemberIterator(&members[mit->second]);
            }
        }
        # 返回成员结束迭代器
        return MemberEnd();
    }

    # 清空所有成员
    void DoClearMembers() {
        # 如果成员指针存在
        if (Member* members = GetMembersPointer()) {
            # 获取成员对应的映射
            Map* &map = GetMap(members);
            # 获取映射的迭代器数组
            MapIterator* mit = GetMapIterators(map);
            # 遍历所有成员
            for (SizeType i = 0; i < data_.o.size; i++) {
                # 从映射中删除对应的迭代器
                map->erase(DropMapIterator(mit[i]));
                # 销毁成员对象
                members[i].~Member();
            }
            # 重置成员数量为0
            data_.o.size = 0;
        }
    }

    # 释放所有成员
    void DoFreeMembers() {
        # 如果成员指针存在
        if (Member* members = GetMembersPointer()) {
            # 销毁成员对应的映射
            GetMap(members)->~Map();
            # 遍历所有成员
            for (SizeType i = 0; i < data_.o.size; i++) {
                # 销毁成员对象
                members[i].~Member();
            }
            # 如果需要释放内存
            if (Allocator::kNeedFree) { 
                # 释放映射指针
                Map** map = &GetMap(members);
                Allocator::Free(*map);
                # 释放映射指针的内存
                Allocator::Free(map);
            }
        }
    }
#else // !RAPIDJSON_USE_MEMBERSMAP

    // 分配成员数组的内存空间
    RAPIDJSON_FORCEINLINE Member* DoAllocMembers(SizeType capacity, Allocator& allocator) {
        return Malloc<Member>(allocator, capacity);
    }

    // 重新分配成员数组的内存空间
    void DoReserveMembers(SizeType newCapacity, Allocator& allocator) {
        ObjectData& o = data_.o;
        if (newCapacity > o.capacity) {
            Member* newMembers = Realloc<Member>(allocator, GetMembersPointer(), o.capacity, newCapacity);
            RAPIDJSON_SETPOINTER(Member, o.members, newMembers);
            o.capacity = newCapacity;
        }
    }

    // 查找指定名称的成员
    template <typename SourceAllocator>
    MemberIterator DoFindMember(const GenericValue<Encoding, SourceAllocator>& name) {
        MemberIterator member = MemberBegin();
        for ( ; member != MemberEnd(); ++member)
            if (name.StringEqual(member->name))
                break;
        return member;
    }

    // 清空成员数组
    void DoClearMembers() {
        for (MemberIterator m = MemberBegin(); m != MemberEnd(); ++m)
            m->~Member();
        data_.o.size = 0;
    }

    // 释放成员数组的内存空间
    void DoFreeMembers() {
        for (MemberIterator m = MemberBegin(); m != MemberEnd(); ++m)
            m->~Member();
        Allocator::Free(GetMembersPointer());
    }

#endif // !RAPIDJSON_USE_MEMBERSMAP

    // 向对象中添加成员
    void DoAddMember(GenericValue& name, GenericValue& value, Allocator& allocator) {
        ObjectData& o = data_.o;
        if (o.size >= o.capacity)
            DoReserveMembers(o.capacity ? (o.capacity + (o.capacity + 1) / 2) : kDefaultObjectCapacity, allocator);
        Member* members = GetMembersPointer();
        Member* m = members + o.size;
        m->name.RawAssign(name);
        m->value.RawAssign(value);
#if RAPIDJSON_USE_MEMBERSMAP
        Map* &map = GetMap(members);
        MapIterator* mit = GetMapIterators(map);
        new (&mit[o.size]) MapIterator(map->insert(MapPair(m->name.data_, o.size)));
#endif
        ++o.size;
    }
        # 从成员迭代器中移除成员，并返回移除后的迭代器
        # 获取数据对象的引用
        ObjectData& o = data_.o;
        # 获取成员指针
        Member* members = GetMembersPointer();
#if RAPIDJSON_USE_MEMBERSMAP
        // 如果使用了成员映射，则获取成员映射指针，并获取成员映射迭代器
        Map* &map = GetMap(members);
        MapIterator* mit = GetMapIterators(map);
        // 计算当前成员在成员映射中的位置，并从映射中删除对应的迭代器
        SizeType mpos = static_cast<SizeType>(&*m - members);
        map->erase(DropMapIterator(mit[mpos]));
#endif
        // 获取最后一个成员的迭代器
        MemberIterator last(members + (o.size - 1));
        // 如果对象中有多个成员且当前成员不是最后一个
        if (o.size > 1 && m != last) {
#if RAPIDJSON_USE_MEMBERSMAP
            // 如果使用了成员映射，则更新成员映射迭代器，并更新映射中对应的位置
            new (&mit[mpos]) MapIterator(DropMapIterator(mit[&*last - members]));
            mit[mpos]->second = mpos;
#endif
            // 将最后一个成员移动到当前位置
            *m = *last; // Move the last one to this place
        }
        else {
            // 如果只剩下一个成员，则销毁当前成员
            m->~Member(); // Only one left, just destroy
        }
        // 减少对象中成员的数量
        --o.size;
        // 返回当前成员的迭代器
        return m;
    }

    MemberIterator DoEraseMembers(ConstMemberIterator first, ConstMemberIterator last) {
        ObjectData& o = data_.o;
        // 获取对象中成员的起始迭代器、指定位置的迭代器和结束迭代器
        MemberIterator beg = MemberBegin(),
                       pos = beg + (first - beg),
                       end = MemberEnd();
#if RAPIDJSON_USE_MEMBERSMAP
        // 如果使用了成员映射，则获取成员映射指针，并获取成员映射迭代器
        Map* &map = GetMap(GetMembersPointer());
        MapIterator* mit = GetMapIterators(map);
#endif
        // 遍历要删除的成员范围
        for (MemberIterator itr = pos; itr != last; ++itr) {
#if RAPIDJSON_USE_MEMBERSMAP
            // 如果使用了成员映射，则从映射中删除对应的迭代器
            map->erase(DropMapIterator(mit[itr - beg]));
#endif
            // 销毁当前成员
            itr->~Member();
        }
#if RAPIDJSON_USE_MEMBERSMAP
        // 如果使用了成员映射，并且要删除的范围不为空
        if (first != last) {
            // 移动剩余的成员/迭代器
            MemberIterator next = pos + (last - first);
            for (MemberIterator itr = pos; next != end; ++itr, ++next) {
                // 使用内存拷贝将下一个成员的数据复制到当前位置
                std::memcpy(static_cast<void*>(&*itr), &*next, sizeof(Member));
                // 计算当前成员在成员映射中的位置，并更新成员映射中对应的位置
                SizeType mpos = static_cast<SizeType>(itr - beg);
                new (&mit[mpos]) MapIterator(DropMapIterator(mit[next - beg]));
                mit[mpos]->second = mpos;
            }
        }
#else
        // 如果没有使用成员映射，则使用内存移动将剩余的成员数据移动到指定位置
        std::memmove(static_cast<void*>(&*pos), &*last,
                     static_cast<size_t>(end - last) * sizeof(Member));
#endif
        // 减去指定范围的大小
        o.size -= static_cast<SizeType>(last - first);
        // 返回位置
        return pos;
    }

    template <typename SourceAllocator>
    void DoCopyMembers(const GenericValue<Encoding,SourceAllocator>& rhs, Allocator& allocator, bool copyConstStrings) {
        // 断言传入的值是对象类型
        RAPIDJSON_ASSERT(rhs.GetType() == kObjectType);

        // 设置标志为对象类型
        data_.f.flags = kObjectFlag;
        // 获取成员数量
        SizeType count = rhs.data_.o.size;
        // 分配内存给成员
        Member* lm = DoAllocMembers(count, allocator);
        // 获取右值的成员指针
        const typename GenericValue<Encoding,SourceAllocator>::Member* rm = rhs.GetMembersPointer();
#if RAPIDJSON_USE_MEMBERSMAP
        // 获取成员映射
        Map* &map = GetMap(lm);
        // 获取成员映射迭代器
        MapIterator* mit = GetMapIterators(map);
#endif
        // 遍历成员
        for (SizeType i = 0; i < count; i++) {
            // 在分配的内存上构造成员的名称
            new (&lm[i].name) GenericValue(rm[i].name, allocator, copyConstStrings);
            // 在分配的内存上构造成员的值
            new (&lm[i].value) GenericValue(rm[i].value, allocator, copyConstStrings);
#if RAPIDJSON_USE_MEMBERSMAP
            // 在成员映射迭代器上构造映射
            new (&mit[i]) MapIterator(map->insert(MapPair(lm[i].name.data_, i)));
#endif
        }
        // 设置成员数量和容量
        data_.o.size = data_.o.capacity = count;
        // 设置成员指针
        SetMembersPointer(lm);
    }

    // 以初始数据初始化数组，不调用析构函数
    void SetArrayRaw(GenericValue* values, SizeType count, Allocator& allocator) {
        // 设置标志为数组类型
        data_.f.flags = kArrayFlag;
        if (count) {
            // 分配内存给数组元素
            GenericValue* e = static_cast<GenericValue*>(allocator.Malloc(count * sizeof(GenericValue)));
            // 设置元素指针
            SetElementsPointer(e);
            // 复制初始数据到分配的内存
            std::memcpy(static_cast<void*>(e), values, count * sizeof(GenericValue));
        }
        else
            // 如果数量为0，则设置元素指针为0
            SetElementsPointer(0);
        // 设置数组大小和容量
        data_.a.size = data_.a.capacity = count;
    }

    //! 以初始数据初始化对象，不调用析构函数。
        // 设置对象的标志位为 kObjectFlag
        data_.f.flags = kObjectFlag;
        // 如果成员数量不为零
        if (count) {
            // 分配内存空间给成员数组
            Member* m = DoAllocMembers(count, allocator);
            // 设置成员指针
            SetMembersPointer(m);
            // 将 members 数组的内容复制到成员数组中
            std::memcpy(static_cast<void*>(m), members, count * sizeof(Member));
#if RAPIDJSON_USE_MEMBERSMAP
            // 如果定义了RAPIDJSON_USE_MEMBERSMAP，则执行以下代码块
            Map* &map = GetMap(m);
            // 获取m的Map指针
            MapIterator* mit = GetMapIterators(map);
            // 获取map的迭代器
            for (SizeType i = 0; i < count; i++) {
                // 遍历count次
                new (&mit[i]) MapIterator(map->insert(MapPair(m[i].name.data_, i)));
                // 在mit[i]处构造MapIterator对象，插入map中
            }
#endif
        }
        else
            // 如果未定义RAPIDJSON_USE_MEMBERSMAP，则执行以下代码块
            SetMembersPointer(0);
            // 设置成员指针为0
        data_.o.size = data_.o.capacity = count;
        // 设置data_.o.size和data_.o.capacity为count

    }

    //! Initialize this value as constant string, without calling destructor.
    void SetStringRaw(StringRefType s) RAPIDJSON_NOEXCEPT {
        // 以常量字符串形式初始化该值，不调用析构函数
        data_.f.flags = kConstStringFlag;
        // 设置data_.f.flags为kConstStringFlag
        SetStringPointer(s);
        // 设置字符串指针为s
        data_.s.length = s.length;
        // 设置data_.s.length为s.length
    }

    //! Initialize this value as copy string with initial data, without calling destructor.
    void SetStringRaw(StringRefType s, Allocator& allocator) {
        // 以初始数据的复制字符串形式初始化该值，不调用析构函数
        Ch* str = 0;
        // 初始化Ch* str为0
        if (ShortString::Usable(s.length)) {
            // 如果s.length可用
            data_.f.flags = kShortStringFlag;
            // 设置data_.f.flags为kShortStringFlag
            data_.ss.SetLength(s.length);
            // 设置data_.ss的长度为s.length
            str = data_.ss.str;
            // 设置str为data_.ss.str
        } else {
            // 如果s.length不可用
            data_.f.flags = kCopyStringFlag;
            // 设置data_.f.flags为kCopyStringFlag
            data_.s.length = s.length;
            // 设置data_.s.length为s.length
            str = static_cast<Ch *>(allocator.Malloc((s.length + 1) * sizeof(Ch)));
            // 分配内存给str
            SetStringPointer(str);
            // 设置字符串指针为str
        }
        std::memcpy(str, s, s.length * sizeof(Ch));
        // 复制s的数据到str
        str[s.length] = '\0';
        // 在str的末尾添加'\0'
    }

    //! Assignment without calling destructor
    void RawAssign(GenericValue& rhs) RAPIDJSON_NOEXCEPT {
        // 不调用析构函数的赋值
        data_ = rhs.data_;
        // 将rhs的data_赋值给当前对象的data_
        // data_.f.flags = rhs.data_.f.flags;
        rhs.data_.f.flags = kNullFlag;
        // 将rhs的data_.f.flags设置为kNullFlag
    }

    template <typename SourceAllocator>
    // 比较当前字符串和传入的字符串是否相等
    bool StringEqual(const GenericValue<Encoding, SourceAllocator>& rhs) const {
        // 断言当前值是字符串类型
        RAPIDJSON_ASSERT(IsString());
        // 断言传入的值是字符串类型
        RAPIDJSON_ASSERT(rhs.IsString());

        // 获取当前字符串的长度
        const SizeType len1 = GetStringLength();
        // 获取传入字符串的长度
        const SizeType len2 = rhs.GetStringLength();
        // 如果长度不相等，返回false
        if(len1 != len2) { return false; }

        // 获取当前字符串的指针
        const Ch* const str1 = GetString();
        // 获取传入字符串的指针
        const Ch* const str2 = rhs.GetString();
        // 如果两个字符串指针相等，直接返回true，快速路径处理常量字符串
        if(str1 == str2) { return true; } // fast path for constant string

        // 使用内存比较函数比较两个字符串的内容是否相等
        return (std::memcmp(str1, str2, sizeof(Ch) * len1) == 0);
    }

    // 数据成员
    Data data_;
};

//! 使用 UTF8 编码的 GenericValue
typedef GenericValue<UTF8<> > Value;

///////////////////////////////////////////////////////////////////////////////
// GenericDocument

//! 用于将 JSON 文本解析为 DOM 的文档。
/*!
    \note 实现了 Handler 概念
    \tparam Encoding 用于解析和字符串存储的编码。
    \tparam Allocator 用于为 DOM 分配内存的分配器
    \tparam StackAllocator 用于在解析过程中分配堆栈内存的分配器。
    \warning 虽然 GenericDocument 继承自 GenericValue，但 API \b 不提供任何虚函数，特别是没有虚析构函数。为了避免内存泄漏，不要通过指向 GenericValue 的指针 \c delete 一个 GenericDocument 对象。
*/
template <typename Encoding, typename Allocator = RAPIDJSON_DEFAULT_ALLOCATOR, typename StackAllocator = RAPIDJSON_DEFAULT_STACK_ALLOCATOR >
class GenericDocument : public GenericValue<Encoding, Allocator> {
public:
    typedef typename Encoding::Ch Ch;                       //!< 从编码派生的字符类型。
    typedef GenericValue<Encoding, Allocator> ValueType;    //!< 文档的值类型。
    typedef Allocator AllocatorType;                        //!< 模板参数中的分配器类型。

    //! 构造函数
    /*! 创建指定类型的空文档。
        \param type             必须创建的对象类型。
        \param allocator        用于分配内存的可选分配器。
        \param stackCapacity    可选的堆栈初始容量（以字节为单位）。
        \param stackAllocator   用于分配堆栈内存的可选分配器。
    */
    explicit GenericDocument(Type type, Allocator* allocator = 0, size_t stackCapacity = kDefaultStackCapacity, StackAllocator* stackAllocator = 0) :
        GenericValue<Encoding, Allocator>(type),  allocator_(allocator), ownAllocator_(0), stack_(stackAllocator, stackCapacity), parseResult_()
    {
        // 如果当前的 allocator_ 为空指针，则创建一个新的 Allocator 对象并赋值给 allocator_
        if (!allocator_)
            ownAllocator_ = allocator_ = RAPIDJSON_NEW(Allocator)();
    }

    //! Constructor
    /*! 创建一个空的文档，类型为 Null。
        \param allocator        用于分配内存的可选分配器。
        \param stackCapacity    可选的堆栈初始容量（以字节为单位）。
        \param stackAllocator   用于为堆栈分配内存的可选分配器。
    */
    GenericDocument(Allocator* allocator = 0, size_t stackCapacity = kDefaultStackCapacity, StackAllocator* stackAllocator = 0) :
        allocator_(allocator), ownAllocator_(0), stack_(stackAllocator, stackCapacity), parseResult_()
    {
        // 如果当前的 allocator_ 为空指针，则创建一个新的 Allocator 对象并赋值给 allocator_
        if (!allocator_)
            ownAllocator_ = allocator_ = RAPIDJSON_NEW(Allocator)();
    }
#if RAPIDJSON_HAS_CXX11_RVALUE_REFS
    //! 如果支持 C++11 的右值引用，则定义移动构造函数
    GenericDocument(GenericDocument&& rhs) RAPIDJSON_NOEXCEPT
        : ValueType(std::forward<ValueType>(rhs)), // 显式转换以避免从 Document 禁止移动
          allocator_(rhs.allocator_),
          ownAllocator_(rhs.ownAllocator_),
          stack_(std::move(rhs.stack_)),
          parseResult_(rhs.parseResult_)
    {
        rhs.allocator_ = 0;
        rhs.ownAllocator_ = 0;
        rhs.parseResult_ = ParseResult();
    }
#endif

    ~GenericDocument() {
        // 在 ownAllocator 被销毁之前清除 ::ValueType，~ValueType() 最后运行，可能访问其元素或成员，这些将使用 MemoryPoolAllocator（CrtAllocator 在销毁时不释放其数据，但 MemoryPoolAllocator 会）。
        if (ownAllocator_) {
            ValueType::SetNull();
        }
        Destroy();
    }

#if RAPIDJSON_HAS_CXX11_RVALUE_REFS
    //! 如果支持 C++11 的右值引用，则定义移动赋值运算符
    GenericDocument& operator=(GenericDocument&& rhs) RAPIDJSON_NOEXCEPT
    {
        // 这里对 ValueType 进行转换是必要的，否则它会尝试调用 GenericValue 的模板赋值运算符。
        ValueType::operator=(std::forward<ValueType>(rhs));

        // 在这里调用析构函数会过早调用 stack_ 的析构函数
        Destroy();

        allocator_ = rhs.allocator_;
        ownAllocator_ = rhs.ownAllocator_;
        stack_ = std::move(rhs.stack_);
        parseResult_ = rhs.parseResult_;

        rhs.allocator_ = 0;
        rhs.ownAllocator_ = 0;
        rhs.parseResult_ = ParseResult();

        return *this;
    }
#endif

    //! 交换该文档的内容与另一个文档的内容。
    /*!
        \param rhs 另一个文档。
        \note 常量复杂度。
        \see GenericValue::Swap
    */
    // 通过交换操作，将当前文档对象与另一个文档对象进行交换
    GenericDocument& Swap(GenericDocument& rhs) RAPIDJSON_NOEXCEPT {
        // 调用基类的 Swap 方法
        ValueType::Swap(rhs);
        // 交换当前文档对象的堆栈与另一个文档对象的堆栈
        stack_.Swap(rhs.stack_);
        // 交换当前文档对象的分配器与另一个文档对象的分配器
        internal::Swap(allocator_, rhs.allocator_);
        // 交换当前文档对象的拥有分配器标记与另一个文档对象的拥有分配器标记
        internal::Swap(ownAllocator_, rhs.ownAllocator_);
        // 交换当前文档对象的解析结果与另一个文档对象的解析结果
        internal::Swap(parseResult_, rhs.parseResult_);
        // 返回当前文档对象的引用
        return *this;
    }

    // 允许与 ValueType 进行交换
    // 参考 Effective C++ 3rd Edition/Item 33: Avoid hiding inherited names.
    using ValueType::Swap;

    //! 自由的交换函数助手
    /*!
        辅助函数，用于支持基于 \c std::swap 的常见交换实现模式：
        \code
        void swap(MyClass& a, MyClass& b) {
            using std::swap;
            swap(a.doc, b.doc);
            // ...
        }
        \endcode
        \see Swap()
     */
    friend inline void swap(GenericDocument& a, GenericDocument& b) RAPIDJSON_NOEXCEPT { a.Swap(b); }

    //! 通过生成器填充文档，生成器产生 SAX 事件
    /*! \tparam Generator 具有 <tt>bool f(Handler)</tt> 原型的函数对象。
        \param g 生成器函数对象，将 SAX 事件发送到参数中。
        \return 用于流畅 API 的文档本身。
    */
    template <typename Generator>
    GenericDocument& Populate(Generator& g) {
        // 在退出时清空堆栈
        ClearStackOnExit scope(*this);
        // 如果生成器函数返回 true
        if (g(*this)) {
            // 断言堆栈大小等于 ValueType 的大小：得到一个且仅得到一个根对象
            RAPIDJSON_ASSERT(stack_.GetSize() == sizeof(ValueType)); 
            // 将值从堆栈移动到文档中
            ValueType::operator=(*stack_.template Pop<ValueType>(1));
        }
        // 返回文档本身
        return *this;
    }

    //!@name 从流中解析
    //!@{

    //! 从输入流中解析 JSON 文本（带编码转换）
    /*! \tparam parseFlags Combination of \ref ParseFlag.
        \tparam SourceEncoding Encoding of input stream
        \tparam InputStream Type of input stream, implementing Stream concept
        \param is Input stream to be parsed.
        \return The document itself for fluent API.
    */
    template <unsigned parseFlags, typename SourceEncoding, typename InputStream>
    GenericDocument& ParseStream(InputStream& is) {
        // 创建一个 JSON 解析器对象，使用指定的编码和分配器
        GenericReader<SourceEncoding, Encoding, StackAllocator> reader(
            stack_.HasAllocator() ? &stack_.GetAllocator() : 0);
        // 在函数退出时清空堆栈
        ClearStackOnExit scope(*this);
        // 使用解析器解析输入流，将结果保存到解析结果中
        parseResult_ = reader.template Parse<parseFlags>(is, *this);
        // 如果解析成功
        if (parseResult_) {
            // 断言堆栈中只有一个根对象
            RAPIDJSON_ASSERT(stack_.GetSize() == sizeof(ValueType)); // Got one and only one root object
            // 将堆栈中的值移动到文档中
            ValueType::operator=(*stack_.template Pop<ValueType>(1));// Move value from stack to document
        }
        // 返回文档对象
        return *this;
    }

    //! Parse JSON text from an input stream
    /*! \tparam parseFlags Combination of \ref ParseFlag.
        \tparam InputStream Type of input stream, implementing Stream concept
        \param is Input stream to be parsed.
        \return The document itself for fluent API.
    */
    template <unsigned parseFlags, typename InputStream>
    GenericDocument& ParseStream(InputStream& is) {
        // 调用带模板参数的 ParseStream 函数
        return ParseStream<parseFlags, Encoding, InputStream>(is);
    }

    //! Parse JSON text from an input stream (with \ref kParseDefaultFlags)
    /*! \tparam InputStream Type of input stream, implementing Stream concept
        \param is Input stream to be parsed.
        \return The document itself for fluent API.
    */
    template <typename InputStream>
    GenericDocument& ParseStream(InputStream& is) {
        // 调用带默认解析标志的 ParseStream 函数
        return ParseStream<kParseDefaultFlags, Encoding, InputStream>(is);
    }
    //!@}

    //!@name Parse in-place from mutable string
    //!@{

    //! Parse JSON text from a mutable string
    /*! \tparam parseFlags Combination of \ref ParseFlag.
        \param str Mutable zero-terminated string to be parsed.
        \return The document itself for fluent API.
    */
    template <unsigned parseFlags>
    GenericDocument& ParseInsitu(Ch* str) {
        // 创建一个可变的零结尾字符串流
        GenericInsituStringStream<Encoding> s(str);
        // 调用ParseStream函数解析JSON文本，并使用kParseInsituFlag标志
        return ParseStream<parseFlags | kParseInsituFlag>(s);
    }
    
    //! Parse JSON text from a mutable string (with \ref kParseDefaultFlags)
    /*! \param str Mutable zero-terminated string to be parsed.
        \return The document itself for fluent API.
    */
    GenericDocument& ParseInsitu(Ch* str) {
        // 调用ParseInsitu函数，使用kParseDefaultFlags标志
        return ParseInsitu<kParseDefaultFlags>(str);
    }
    //!@}
    
    //!@name Parse from read-only string
    //!@{
    
    //! Parse JSON text from a read-only string (with Encoding conversion)
    /*! \tparam parseFlags Combination of \ref ParseFlag (must not contain \ref kParseInsituFlag).
        \tparam SourceEncoding Transcoding from input Encoding
        \param str Read-only zero-terminated string to be parsed.
    */
    template <unsigned parseFlags, typename SourceEncoding>
    GenericDocument& Parse(const typename SourceEncoding::Ch* str) {
        // 断言parseFlags中不包含kParseInsituFlag标志
        RAPIDJSON_ASSERT(!(parseFlags & kParseInsituFlag));
        // 创建一个只读的零结尾字符串流
        GenericStringStream<SourceEncoding> s(str);
        // 调用ParseStream函数解析JSON文本，使用给定的parseFlags和SourceEncoding
        return ParseStream<parseFlags, SourceEncoding>(s);
    }
    
    //! Parse JSON text from a read-only string
    /*! \tparam parseFlags Combination of \ref ParseFlag (must not contain \ref kParseInsituFlag).
        \param str Read-only zero-terminated string to be parsed.
    */
    template <unsigned parseFlags>
    GenericDocument& Parse(const Ch* str) {
        // 调用Parse函数，使用给定的parseFlags和默认的Encoding
        return Parse<parseFlags, Encoding>(str);
    }
    
    //! Parse JSON text from a read-only string (with \ref kParseDefaultFlags)
    /*! \param str Read-only zero-terminated string to be parsed.
    */
    GenericDocument& Parse(const Ch* str) {
        // 调用Parse函数，使用kParseDefaultFlags标志和默认的Encoding
        return Parse<kParseDefaultFlags>(str);
    }
    // 使用指定的解析标志和源编码解析给定的字符串
    template <unsigned parseFlags, typename SourceEncoding>
    GenericDocument& Parse(const typename SourceEncoding::Ch* str, size_t length) {
        // 断言解析标志中不包含原地解析标志
        RAPIDJSON_ASSERT(!(parseFlags & kParseInsituFlag));
        // 创建内存流对象，将字符串转换为字符流
        MemoryStream ms(reinterpret_cast<const char*>(str), length * sizeof(typename SourceEncoding::Ch));
        // 创建编码输入流对象，使用内存流作为输入
        EncodedInputStream<SourceEncoding, MemoryStream> is(ms);
        // 调用解析流函数，解析输入流中的内容
        ParseStream<parseFlags, SourceEncoding>(is);
        // 返回解析后的文档对象的引用
        return *this;
    }

    // 使用指定的解析标志解析给定的字符串
    template <unsigned parseFlags>
    GenericDocument& Parse(const Ch* str, size_t length) {
        // 调用带解析标志和编码类型的解析函数
        return Parse<parseFlags, Encoding>(str, length);
    }

    // 使用默认解析标志解析给定的字符串
    GenericDocument& Parse(const Ch* str, size_t length) {
        // 调用带默认解析标志的解析函数
        return Parse<kParseDefaultFlags>(str, length);
    }
#if RAPIDJSON_HAS_STDSTRING
    // 如果编译器支持 std::string，则定义模板函数，接受字符串作为输入并解析
    template <unsigned parseFlags, typename SourceEncoding>
    GenericDocument& Parse(const std::basic_string<typename SourceEncoding::Ch>& str) {
        // 使用 c_str() 方法将字符串转换为 const char*，以便进行解析
        // 根据标准，c_str() 的复杂度是常量级别的，应该比 Parse(const char*, size_t) 更快
        return Parse<parseFlags, SourceEncoding>(str.c_str());
    }

    // 如果编译器支持 std::string，则定义模板函数，接受字符串作为输入并解析
    template <unsigned parseFlags>
    GenericDocument& Parse(const std::basic_string<Ch>& str) {
        // 调用 Parse 函数进行解析
        return Parse<parseFlags, Encoding>(str.c_str());
    }

    // 定义函数，接受字符串作为输入并解析
    GenericDocument& Parse(const std::basic_string<Ch>& str) {
        // 调用 Parse 函数进行解析，使用默认的解析标志
        return Parse<kParseDefaultFlags>(str);
    }
#endif // RAPIDJSON_HAS_STDSTRING

    //!@}

    //!@name Handling parse errors
    //!@{

    // 检查上次解析是否发生错误
    bool HasParseError() const { return parseResult_.IsError(); }

    // 获取上次解析的错误代码
    ParseErrorCode GetParseError() const { return parseResult_.Code(); }

    // 获取上次解析错误的位置，如果没有错误则返回 0
    size_t GetErrorOffset() const { return parseResult_.Offset(); }

    // 隐式转换，获取上次解析的结果
#ifndef __clang // -Wdocumentation
    /*! \return \ref ParseResult of the last parse operation

        \code
          Document doc;
          ParseResult ok = doc.Parse(json);
          if (!ok)
            printf( "JSON parse error: %s (%u)\n", GetParseError_En(ok.Code()), ok.Offset());
        \endcode
     */
#endif
    operator ParseResult() const { return parseResult_; }
    //!@}

    // 获取文档的分配器
    Allocator& GetAllocator() {
        RAPIDJSON_ASSERT(allocator_);
        return *allocator_;
    }

    // 获取堆栈的容量（以字节为单位）
    size_t GetStackCapacity() const { return stack_.GetCapacity(); }

private:
    // 在从 ParseStream 中的任何退出时清除堆栈，例如由于异常引起的退出
    // 定义一个结构体，用于在退出时清空栈
    struct ClearStackOnExit {
        // 显式构造函数，接受一个 GenericDocument 的引用
        explicit ClearStackOnExit(GenericDocument& d) : d_(d) {}
        // 析构函数，在退出时调用 GenericDocument 的 ClearStack 方法
        ~ClearStackOnExit() { d_.ClearStack(); }
    private:
        // 禁止复制构造函数
        ClearStackOnExit(const ClearStackOnExit&);
        // 禁止赋值运算符重载
        ClearStackOnExit& operator=(const ClearStackOnExit&);
        // 保存 GenericDocument 的引用
        GenericDocument& d_;
    };

    // 以下是私有 Handler 函数的调用者
    // 用于解析的 GenericReader 的模板类，作为友元类
    // template <typename,typename,typename> friend class GenericReader; // for parsing
    // 用于深度复制的 GenericValue 的模板类，作为友元类
    template <typename, typename> friend class GenericValue; // for deep copying
public:
    // 实现 Handler 接口，创建一个空值并压入栈中
    bool Null() { new (stack_.template Push<ValueType>()) ValueType(); return true; }
    // 实现 Handler 接口，创建一个布尔值并压入栈中
    bool Bool(bool b) { new (stack_.template Push<ValueType>()) ValueType(b); return true; }
    // 实现 Handler 接口，创建一个整数值并压入栈中
    bool Int(int i) { new (stack_.template Push<ValueType>()) ValueType(i); return true; }
    // 实现 Handler 接口，创建一个无符号整数值并压入栈中
    bool Uint(unsigned i) { new (stack_.template Push<ValueType>()) ValueType(i); return true; }
    // 实现 Handler 接口，创建一个64位整数值并压入栈中
    bool Int64(int64_t i) { new (stack_.template Push<ValueType>()) ValueType(i); return true; }
    // 实现 Handler 接口，创建一个64位无符号整数值并压入栈中
    bool Uint64(uint64_t i) { new (stack_.template Push<ValueType>()) ValueType(i); return true; }
    // 实现 Handler 接口，创建一个双精度浮点数值并压入栈中
    bool Double(double d) { new (stack_.template Push<ValueType>()) ValueType(d); return true; }

    // 实现 Handler 接口，根据参数决定是否复制字符串并压入栈中
    bool RawNumber(const Ch* str, SizeType length, bool copy) {
        if (copy)
            new (stack_.template Push<ValueType>()) ValueType(str, length, GetAllocator());
        else
            new (stack_.template Push<ValueType>()) ValueType(str, length);
        return true;
    }

    // 实现 Handler 接口，根据参数决定是否复制字符串并压入栈中
    bool String(const Ch* str, SizeType length, bool copy) {
        if (copy)
            new (stack_.template Push<ValueType>()) ValueType(str, length, GetAllocator());
        else
            new (stack_.template Push<ValueType>()) ValueType(str, length);
        return true;
    }

    // 实现 Handler 接口，创建一个对象值并压入栈中
    bool StartObject() { new (stack_.template Push<ValueType>()) ValueType(kObjectType); return true; }

    // 实现 Handler 接口，调用 String 方法
    bool Key(const Ch* str, SizeType length, bool copy) { return String(str, length, copy); }

    // 实现 Handler 接口，根据成员数量弹出栈中的成员并设置对象值
    bool EndObject(SizeType memberCount) {
        typename ValueType::Member* members = stack_.template Pop<typename ValueType::Member>(memberCount);
        stack_.template Top<ValueType>()->SetObjectRaw(members, memberCount, GetAllocator());
        return true;
    }

    // 实现 Handler 接口，创建一个数组值并压入栈中
    bool StartArray() { new (stack_.template Push<ValueType>()) ValueType(kArrayType); return true; }
    # 结束数组的解析，并返回布尔值
    bool EndArray(SizeType elementCount) {
        # 从堆栈中弹出指定数量的元素，并将它们存储在 elements 变量中
        ValueType* elements = stack_.template Pop<ValueType>(elementCount);
        # 将 elements 中的元素以原始数组的形式设置到堆栈顶部的值中
        stack_.template Top<ValueType>()->SetArrayRaw(elements, elementCount, GetAllocator());
        # 返回 true，表示数组解析结束
        return true;
    }
private:
    //! 禁止复制
    GenericDocument(const GenericDocument&);
    //! 禁止赋值
    GenericDocument& operator=(const GenericDocument&);

    void ClearStack() {
        if (Allocator::kNeedFree)
            while (stack_.GetSize() > 0)    // 这里假设栈数组中的所有元素都是 GenericValue（实际上是 2 个 GenericValue 对象）
                (stack_.template Pop<ValueType>(1))->~ValueType();
        else
            stack_.Clear();
        stack_.ShrinkToFit();
    }

    void Destroy() {
        RAPIDJSON_DELETE(ownAllocator_);
    }

    static const size_t kDefaultStackCapacity = 1024;
    Allocator* allocator_;
    Allocator* ownAllocator_;
    internal::Stack<StackAllocator> stack_;
    ParseResult parseResult_;
};

//! 使用 UTF8 编码的 GenericDocument
typedef GenericDocument<UTF8<> > Document;


//! 用于访问数组类型值的辅助类。
/*!
    通过 \c GenericValue::GetArray() 获取此辅助类的实例。
    除了数组类型的所有 API 外，如果 \c RAPIDJSON_HAS_CXX11_RANGE_FOR=1，则提供基于范围的 for 循环。
*/
template <bool Const, typename ValueT>
class GenericArray {
public:
    typedef GenericArray<true, ValueT> ConstArray;
    typedef GenericArray<false, ValueT> Array;
    typedef ValueT PlainType;
    typedef typename internal::MaybeAddConst<Const,PlainType>::Type ValueType;
    typedef ValueType* ValueIterator;  // 这可能是 const 或非 const 迭代器
    typedef const ValueT* ConstValueIterator;
    typedef typename ValueType::AllocatorType AllocatorType;
    typedef typename ValueType::StringRefType StringRefType;

    template <typename, typename>
    friend class GenericValue;

    GenericArray(const GenericArray& rhs) : value_(rhs.value_) {}
    GenericArray& operator=(const GenericArray& rhs) { value_ = rhs.value_; return *this; }
    ~GenericArray() {}

    operator ValueType&() const { return value_; }
    SizeType Size() const { return value_.Size(); }
    # 返回数组的容量
    SizeType Capacity() const { return value_.Capacity(); }
    # 检查数组是否为空
    bool Empty() const { return value_.Empty(); }
    # 清空数组
    void Clear() const { value_.Clear(); }
    # 重载操作符，返回指定索引的值的引用
    ValueType& operator[](SizeType index) const {  return value_[index]; }
    # 返回数组的起始迭代器
    ValueIterator Begin() const { return value_.Begin(); }
    # 返回数组的结束迭代器
    ValueIterator End() const { return value_.End(); }
    # 为数组预留新的容量
    GenericArray Reserve(SizeType newCapacity, AllocatorType &allocator) const { value_.Reserve(newCapacity, allocator); return *this; }
    # 将值添加到数组的末尾
    GenericArray PushBack(ValueType& value, AllocatorType& allocator) const { value_.PushBack(value, allocator); return *this; }
#if RAPIDJSON_HAS_CXX11_RVALUE_REFS
    // 如果支持 C++11 的右值引用，则定义 PushBack 方法，将值移动到数组中，并返回数组对象的引用
    GenericArray PushBack(ValueType&& value, AllocatorType& allocator) const { value_.PushBack(value, allocator); return *this; }
#endif // RAPIDJSON_HAS_CXX11_RVALUE_REFS
    // 定义 PushBack 方法，将字符串值添加到数组中，并返回数组对象的引用
    GenericArray PushBack(StringRefType value, AllocatorType& allocator) const { value_.PushBack(value, allocator); return *this; }
    // 定义 PushBack 方法的模板，将值添加到数组中，并返回数组对象的引用
    template <typename T> RAPIDJSON_DISABLEIF_RETURN((internal::OrExpr<internal::IsPointer<T>, internal::IsGenericValue<T> >), (const GenericArray&)) PushBack(T value, AllocatorType& allocator) const { value_.PushBack(value, allocator); return *this; }
    // 定义 PopBack 方法，从数组中弹出最后一个值，并返回数组对象的引用
    GenericArray PopBack() const { value_.PopBack(); return *this; }
    // 定义 Erase 方法，从数组中删除指定位置的值，并返回删除后的数组对象的引用
    ValueIterator Erase(ConstValueIterator pos) const { return value_.Erase(pos); }
    // 定义 Erase 方法，从数组中删除指定范围的值，并返回删除后的数组对象的引用
    ValueIterator Erase(ConstValueIterator first, ConstValueIterator last) const { return value_.Erase(first, last); }

#if RAPIDJSON_HAS_CXX11_RANGE_FOR
    // 如果支持 C++11 的范围 for 循环，则定义 begin 方法，返回数组对象的起始迭代器
    ValueIterator begin() const { return value_.Begin(); }
    // 如果支持 C++11 的范围 for 循环，则定义 end 方法，返回数组对象的结束迭代器
    ValueIterator end() const { return value_.End(); }
#endif

private:
    // 私有构造函数，禁止外部创建对象
    GenericArray();
    // 带参构造函数，初始化数组对象的值
    GenericArray(ValueType& value) : value_(value) {}
    // 数组对象的值
    ValueType& value_;
};

//! Helper class for accessing Value of object type.
/*!
    Instance of this helper class is obtained by \c GenericValue::GetObject().
    In addition to all APIs for array type, it provides range-based for loop if \c RAPIDJSON_HAS_CXX11_RANGE_FOR=1.
*/
// 用于访问对象类型值的辅助类
template <bool Const, typename ValueT>
class GenericObject {
public:
    // 定义常量对象类型
    typedef GenericObject<true, ValueT> ConstObject;
    // 定义非常量对象类型
    typedef GenericObject<false, ValueT> Object;
    // 定义原始类型
    typedef ValueT PlainType;
    // 定义值类型
    typedef typename internal::MaybeAddConst<Const,PlainType>::Type ValueType;
    // 定义成员迭代器类型，可能是常量或非常量迭代器
    typedef GenericMemberIterator<Const, typename ValueT::EncodingType, typename ValueT::AllocatorType> MemberIterator;  // This may be const or non-const iterator
    // 定义常量成员迭代器类型
    typedef GenericMemberIterator<true, typename ValueT::EncodingType, typename ValueT::AllocatorType> ConstMemberIterator;
    // 定义类型别名，用于简化模板类型的使用
    typedef typename ValueType::AllocatorType AllocatorType;
    typedef typename ValueType::StringRefType StringRefType;
    typedef typename ValueType::EncodingType EncodingType;
    typedef typename ValueType::Ch Ch;

    // 声明 GenericValue 类为 GenericObject 的友元类
    template <typename, typename>
    friend class GenericValue;

    // 拷贝构造函数，使用 rhs 的 value_ 进行初始化
    GenericObject(const GenericObject& rhs) : value_(rhs.value_) {}

    // 赋值运算符重载，将 rhs 的 value_ 赋值给当前对象的 value_
    GenericObject& operator=(const GenericObject& rhs) { value_ = rhs.value_; return *this; }

    // 析构函数，无需执行任何操作
    ~GenericObject() {}

    // 类型转换运算符重载，将当前对象转换为 ValueType 的引用
    operator ValueType&() const { return value_; }

    // 返回对象成员的数量
    SizeType MemberCount() const { return value_.MemberCount(); }

    // 返回对象成员的容量
    SizeType MemberCapacity() const { return value_.MemberCapacity(); }

    // 判断对象是否为空
    bool ObjectEmpty() const { return value_.ObjectEmpty(); }

    // 重载操作符，根据名称获取对象成员的值
    template <typename T> ValueType& operator[](T* name) const { return value_[name]; }

    // 重载操作符，根据 GenericValue 对象获取对象成员的值
    template <typename SourceAllocator> ValueType& operator[](const GenericValue<EncodingType, SourceAllocator>& name) const { return value_[name]; }
#if RAPIDJSON_HAS_STDSTRING
    // 如果定义了RAPIDJSON_HAS_STDSTRING，则重载[]操作符，返回对应名称的值
    ValueType& operator[](const std::basic_string<Ch>& name) const { return value_[name]; }
#endif
    // 返回成员的起始迭代器
    MemberIterator MemberBegin() const { return value_.MemberBegin(); }
    // 返回成员的结束迭代器
    MemberIterator MemberEnd() const { return value_.MemberEnd(); }
    // 预留给定容量的成员空间
    GenericObject MemberReserve(SizeType newCapacity, AllocatorType &allocator) const { value_.MemberReserve(newCapacity, allocator); return *this; }
    // 检查是否存在指定名称的成员
    bool HasMember(const Ch* name) const { return value_.HasMember(name); }
#if RAPIDJSON_HAS_STDSTRING
    // 如果定义了RAPIDJSON_HAS_STDSTRING，则重载HasMember函数，检查是否存在指定名称的成员
    bool HasMember(const std::basic_string<Ch>& name) const { return value_.HasMember(name); }
#endif
    // 检查是否存在指定名称的成员
    template <typename SourceAllocator> bool HasMember(const GenericValue<EncodingType, SourceAllocator>& name) const { return value_.HasMember(name); }
    // 查找指定名称的成员，并返回其迭代器
    MemberIterator FindMember(const Ch* name) const { return value_.FindMember(name); }
    // 查找指定名称的成员，并返回其迭代器
    template <typename SourceAllocator> MemberIterator FindMember(const GenericValue<EncodingType, SourceAllocator>& name) const { return value_.FindMember(name); }
#if RAPIDJSON_HAS_STDSTRING
    // 如果定义了RAPIDJSON_HAS_STDSTRING，则重载FindMember函数，查找指定名称的成员，并返回其迭代器
    MemberIterator FindMember(const std::basic_string<Ch>& name) const { return value_.FindMember(name); }
#endif
    // 添加指定名称和值的成员
    GenericObject AddMember(ValueType& name, ValueType& value, AllocatorType& allocator) const { value_.AddMember(name, value, allocator); return *this; }
    // 添加指定名称和字符串值的成员
    GenericObject AddMember(ValueType& name, StringRefType value, AllocatorType& allocator) const { value_.AddMember(name, value, allocator); return *this; }
#if RAPIDJSON_HAS_STDSTRING
    // 如果定义了RAPIDJSON_HAS_STDSTRING，则重载AddMember函数，添加指定名称和字符串值的成员
    GenericObject AddMember(ValueType& name, std::basic_string<Ch>& value, AllocatorType& allocator) const { value_.AddMember(name, value, allocator); return *this; }
#endif
    // 添加指定名称和值的成员
    template <typename T> RAPIDJSON_DISABLEIF_RETURN((internal::OrExpr<internal::IsPointer<T>, internal::IsGenericValue<T> >), (ValueType&)) AddMember(ValueType& name, T value, AllocatorType& allocator) const { value_.AddMember(name, value, allocator); return *this; }
#if RAPIDJSON_HAS_CXX11_RVALUE_REFS
    # 向 JSON 对象中添加成员，并返回修改后的对象
    GenericObject AddMember(ValueType&& name, ValueType&& value, AllocatorType& allocator) const { value_.AddMember(name, value, allocator); return *this; }
    
    # 向 JSON 对象中添加成员，并返回修改后的对象
    GenericObject AddMember(ValueType&& name, ValueType& value, AllocatorType& allocator) const { value_.AddMember(name, value, allocator); return *this; }
    
    # 向 JSON 对象中添加成员，并返回修改后的对象
    GenericObject AddMember(ValueType& name, ValueType&& value, AllocatorType& allocator) const { value_.AddMember(name, value, allocator); return *this; }
    
    # 向 JSON 对象中添加成员，并返回修改后的对象
    GenericObject AddMember(StringRefType name, ValueType&& value, AllocatorType& allocator) const { value_.AddMember(name, value, allocator); return *this; }
#endif // RAPIDJSON_HAS_CXX11_RVALUE_REFS
    // 向对象中添加成员，传入名称、值和分配器，返回对象本身
    GenericObject AddMember(StringRefType name, ValueType& value, AllocatorType& allocator) const { value_.AddMember(name, value, allocator); return *this; }
    // 向对象中添加成员，传入名称、字符串值和分配器，返回对象本身
    GenericObject AddMember(StringRefType name, StringRefType value, AllocatorType& allocator) const { value_.AddMember(name, value, allocator); return *this; }
    // 向对象中添加成员，传入名称、值和分配器，返回对象本身
    template <typename T> RAPIDJSON_DISABLEIF_RETURN((internal::OrExpr<internal::IsPointer<T>, internal::IsGenericValue<T> >), (GenericObject)) AddMember(StringRefType name, T value, AllocatorType& allocator) const { value_.AddMember(name, value, allocator); return *this; }
    // 移除对象中的所有成员
    void RemoveAllMembers() { value_.RemoveAllMembers(); }
    // 移除对象中指定名称的成员，返回是否成功
    bool RemoveMember(const Ch* name) const { return value_.RemoveMember(name); }
#if RAPIDJSON_HAS_STDSTRING
    // 移除对象中指定名称的成员，返回是否成功
    bool RemoveMember(const std::basic_string<Ch>& name) const { return value_.RemoveMember(name); }
#endif
    // 移除对象中指定名称的成员，返回是否成功
    template <typename SourceAllocator> bool RemoveMember(const GenericValue<EncodingType, SourceAllocator>& name) const { return value_.RemoveMember(name); }
    // 移除对象中指定迭代器指向的成员
    MemberIterator RemoveMember(MemberIterator m) const { return value_.RemoveMember(m); }
    // 通过迭代器位置移除对象中的成员
    MemberIterator EraseMember(ConstMemberIterator pos) const { return value_.EraseMember(pos); }
    // 通过迭代器范围移除对象中的成员
    MemberIterator EraseMember(ConstMemberIterator first, ConstMemberIterator last) const { return value_.EraseMember(first, last); }
    // 通过名称移除对象中的成员，返回是否成功
    bool EraseMember(const Ch* name) const { return value_.EraseMember(name); }
#if RAPIDJSON_HAS_STDSTRING
    // 通过名称移除对象中的成员，返回是否成功
    bool EraseMember(const std::basic_string<Ch>& name) const { return EraseMember(ValueType(StringRef(name))); }
#endif
    // 通过值对象移除对象中的成员，返回是否成功
    template <typename SourceAllocator> bool EraseMember(const GenericValue<EncodingType, SourceAllocator>& name) const { return value_.EraseMember(name); }

#if RAPIDJSON_HAS_CXX11_RANGE_FOR
    // 返回对象成员的起始迭代器
    MemberIterator begin() const { return value_.MemberBegin(); }
    // 返回对象成员的结束迭代器
    MemberIterator end() const { return value_.MemberEnd(); }
#endif

private:
    // 私有构造函数
    GenericObject();
    # 定义一个泛型对象，接受一个值作为参数，并将其存储在成员变量中
    GenericObject(ValueType& value) : value_(value) {}
    # 声明一个引用类型的成员变量，用于存储传入的值
    ValueType& value_;
};

RAPIDJSON_NAMESPACE_END
RAPIDJSON_DIAG_POP

#ifdef RAPIDJSON_WINDOWS_GETOBJECT_WORKAROUND_APPLIED
#pragma pop_macro("GetObject")
#undef RAPIDJSON_WINDOWS_GETOBJECT_WORKAROUND_APPLIED
#endif

#endif // RAPIDJSON_DOCUMENT_H_
```