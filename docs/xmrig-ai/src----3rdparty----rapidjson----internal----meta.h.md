# `xmrig\src\3rdparty\rapidjson\internal\meta.h`

```
// 定义了一个条件为真的布尔类型
template <bool Cond> struct BoolType {
    static const bool Value = Cond;  // 存储条件值
    typedef BoolType Type;  // 定义类型为 BoolType
};
// 定义了一个条件为真的 TrueType 类型
typedef BoolType<true> TrueType;
// 定义了一个条件为假的 FalseType 类型
typedef BoolType<false> FalseType;

// 用于根据条件选择类型的模板
template <bool C> struct SelectIfImpl {
    template <typename T1, typename T2> struct Apply {
        typedef T1 Type;  // 如果条件为真，选择 T1 类型
    };
};
// 特化模板，当条件为假时选择类型
template <> struct SelectIfImpl<false> {
    template <typename T1, typename T2> struct Apply {
        typedef T2 Type;  // 如果条件为假，选择 T2 类型
    };
};
// 根据条件 C，选择 T1 或 T2 类型
template <bool C, typename T1, typename T2> struct SelectIfCond : SelectIfImpl<C>::template Apply<T1,T2> {};
// 根据条件 C，选择 T1 或 T2 类型
template <typename C, typename T1, typename T2> struct SelectIf : SelectIfCond<C::Value, T1, T2> {};

// 逻辑与表达式的条件
template <bool Cond1, bool Cond2> struct AndExprCond : FalseType {};
template <> struct AndExprCond<true, true> : TrueType {};
// 逻辑或表达式的条件
template <bool Cond1, bool Cond2> struct OrExprCond : TrueType {};
template <> struct OrExprCond<false, false> : FalseType {};

// 根据条件 C，选择 TrueType 或 FalseType 类型
template <typename C> struct BoolExpr : SelectIf<C,TrueType,FalseType>::Type {};
// 根据条件 C，选择 TrueType 或 FalseType 类型
template <typename C> struct NotExpr  : SelectIf<C,FalseType,TrueType>::Type {};
// 逻辑与表达式
template <typename C1, typename C2> struct AndExpr : AndExprCond<C1::Value, C2::Value>::Type {};
// 逻辑或表达式
template <typename C1, typename C2> struct OrExpr  : OrExprCond<C1::Value, C2::Value>::Type {};

///////////////////////////////////////////////////////////////////////////////
// AddConst, MaybeAddConst, RemoveConst
// 添加 const 修饰符
template <typename T> struct AddConst { typedef const T Type; };
// 根据条件 Constify，可能添加 const 修饰符
template <bool Constify, typename T> struct MaybeAddConst : SelectIfCond<Constify, const T, T> {};
// 移除 const 修饰符
template <typename T> struct RemoveConst { typedef T Type; };
// 移除 const 修饰符
template <typename T> struct RemoveConst<const T> { typedef T Type; };

///////////////////////////////////////////////////////////////////////////////
// IsSame, IsConst, IsMoreConst, IsPointer
//
// 判断类型 T 和 U 是否相同
template <typename T, typename U> struct IsSame : FalseType {};
template <typename T> struct IsSame<T, T> : TrueType {};

// 判断类型 T 是否带有 const 修饰符
template <typename T> struct IsConst : FalseType {};
template <typename T> struct IsConst<const T> : TrueType {};

// 判断类型 CT 是否比类型 T 更带有 const 修饰符
template <typename CT, typename T>
struct IsMoreConst
    : AndExpr<IsSame<typename RemoveConst<CT>::Type, typename RemoveConst<T>::Type>,
              BoolType<IsConst<CT>::Value >= IsConst<T>::Value> >::Type {};

// 判断类型 T 是否为指针类型
template <typename T> struct IsPointer : FalseType {};
// 判断类型 T 是否为指针类型
template <typename T> struct IsPointer<T*> : TrueType {};
// 如果支持 C++11 的 type_traits，则使用标准库的 is_base_of 判断是否为基类
#if RAPIDJSON_HAS_CXX11_TYPETRAITS

template <typename B, typename D> struct IsBaseOf
    : BoolType< ::std::is_base_of<B,D>::value> {};

#else // 简化版本，采用 Boost 的实现

template<typename B, typename D> struct IsBaseOfImpl {
    // 断言 B 和 D 的大小不为 0
    RAPIDJSON_STATIC_ASSERT(sizeof(B) != 0);
    RAPIDJSON_STATIC_ASSERT(sizeof(D) != 0);

    // 定义 Yes 和 No 类型
    typedef char (&Yes)[1];
    typedef char (&No) [2];

    // 检查 D 是否为 B 的基类
    template <typename T>
    static Yes Check(const D*, T);
    static No  Check(const B*, int);

    // 定义 Host 结构体
    struct Host {
        operator const B*() const;
        operator const D*();
    };

    // 判断 D 是否为 B 的基类
    enum { Value = (sizeof(Check(Host(), 0)) == sizeof(Yes)) };
};

// 判断是否为基类
template <typename B, typename D> struct IsBaseOf
    : OrExpr<IsSame<B, D>, BoolExpr<IsBaseOfImpl<B, D> > >::Type {};

#endif // RAPIDJSON_HAS_CXX11_TYPETRAITS


// EnableIf / DisableIf
//
// 根据条件启用或禁用模板
template <bool Condition, typename T = void> struct EnableIfCond  { typedef T Type; };
template <typename T> struct EnableIfCond<false, T> { /* empty */ };

template <bool Condition, typename T = void> struct DisableIfCond { typedef T Type; };
template <typename T> struct DisableIfCond<true, T> { /* empty */ };

// 根据条件启用或禁用模板
template <typename Condition, typename T = void>
struct EnableIf : EnableIfCond<Condition::Value, T> {};

template <typename Condition, typename T = void>
struct DisableIf : DisableIfCond<Condition::Value, T> {};

// SFINAE（替换失败不是错误）辅助
struct SfinaeTag {};
template <typename T> struct RemoveSfinaeTag;
template <typename T> struct RemoveSfinaeTag<SfinaeTag&(*)(T)> { typedef T Type; };

// 定义宏 RAPIDJSON_REMOVEFPTR_
#define RAPIDJSON_REMOVEFPTR_(type) \
    typename ::RAPIDJSON_NAMESPACE::internal::RemoveSfinaeTag \
        < ::RAPIDJSON_NAMESPACE::internal::SfinaeTag&(*) type>::Type

// 定义宏 RAPIDJSON_ENABLEIF
#define RAPIDJSON_ENABLEIF(cond) \
    # 定义一个类型名为 typename，使用了条件模板参数，根据条件是否满足来选择是否启用该类型
    typename ::RAPIDJSON_NAMESPACE::internal::EnableIf \
        <RAPIDJSON_REMOVEFPTR_(cond)>::Type * = NULL
// 定义一个宏，用于禁用某个条件
#define RAPIDJSON_DISABLEIF(cond) \
    typename ::RAPIDJSON_NAMESPACE::internal::DisableIf \
        <RAPIDJSON_REMOVEFPTR_(cond)>::Type * = NULL

// 定义一个宏，用于在满足条件时启用返回类型
#define RAPIDJSON_ENABLEIF_RETURN(cond,returntype) \
    typename ::RAPIDJSON_NAMESPACE::internal::EnableIf \
        <RAPIDJSON_REMOVEFPTR_(cond), \
         RAPIDJSON_REMOVEFPTR_(returntype)>::Type

// 定义一个宏，用于在不满足条件时禁用返回类型
#define RAPIDJSON_DISABLEIF_RETURN(cond,returntype) \
    typename ::RAPIDJSON_NAMESPACE::internal::DisableIf \
        <RAPIDJSON_REMOVEFPTR_(cond), \
         RAPIDJSON_REMOVEFPTR_(returntype)>::Type

} // namespace internal
RAPIDJSON_NAMESPACE_END
//@endcond

// 如果是 MSC 编译器并且不是 Clang 编译器，则弹出警告
#if defined(_MSC_VER) && !defined(__clang__)
RAPIDJSON_DIAG_POP
#endif

// 如果是 GNU 编译器，则弹出警告
#ifdef __GNUC__
RAPIDJSON_DIAG_POP
#endif

// 结束条件编译
#endif // RAPIDJSON_INTERNAL_META_H_
```