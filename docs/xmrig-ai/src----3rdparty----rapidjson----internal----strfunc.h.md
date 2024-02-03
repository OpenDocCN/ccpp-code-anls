# `xmrig\src\3rdparty\rapidjson\internal\strfunc.h`

```cpp
// 定义了一个宏，用于开始命名空间 rapidjson::internal
#ifndef RAPIDJSON_INTERNAL_STRFUNC_H_
#define RAPIDJSON_INTERNAL_STRFUNC_H_

// 包含 stream.h 头文件
#include "../stream.h"
// 包含 cwchar 头文件
#include <cwchar>

// 命名空间 rapidjson::internal 的开始
RAPIDJSON_NAMESPACE_BEGIN
namespace internal {

//! 自定义的 strlen() 函数，适用于不同的字符类型
/*! \tparam Ch 字符类型（例如 char, wchar_t, short）
    \param s 输入的以 null 结尾的字符串
    \return 字符串中的字符数
    \note 具有与 strlen() 相同的语义，返回值不是 Unicode 代码点的数量
*/
template <typename Ch>
inline SizeType StrLen(const Ch* s) {
    RAPIDJSON_ASSERT(s != 0);
    const Ch* p = s;
    while (*p) ++p;
    return SizeType(p - s);
}

// 重载模板，适用于 char* 类型
template <>
inline SizeType StrLen(const char* s) {
    return SizeType(std::strlen(s));
}

// 重载模板，适用于 wchar_t* 类型
template <>
inline SizeType StrLen(const wchar_t* s) {
    return SizeType(std::wcslen(s));
}

//! 自定义的 strcmpn() 函数，适用于不同的字符类型
/*! \tparam Ch 字符类型（例如 char, wchar_t, short）
    \param s1 输入的以 null 结尾的字符串
    \param s2 输入的以 null 结尾的字符串
    \return 如果相等则返回 0
*/
template<typename Ch>
inline int StrCmp(const Ch* s1, const Ch* s2) {
    RAPIDJSON_ASSERT(s1 != 0);
    RAPIDJSON_ASSERT(s2 != 0);
    while(*s1 && (*s1 == *s2)) { s1++; s2++; }
    # 返回两个字符串指针所指向的字符的无符号整数比较结果
    return static_cast<unsigned>(*s1) < static_cast<unsigned>(*s2) ? -1 : static_cast<unsigned>(*s1) > static_cast<unsigned>(*s2);
// 结束命名空间 internal
}

// 返回编码字符串中的代码点数
template<typename Encoding>
bool CountStringCodePoint(const typename Encoding::Ch* s, SizeType length, SizeType* outCount) {
    // 断言输入字符串不为空
    RAPIDJSON_ASSERT(s != 0);
    // 断言输出计数指针不为空
    RAPIDJSON_ASSERT(outCount != 0);
    // 创建字符串流对象
    GenericStringStream<Encoding> is(s);
    // 计算字符串结束位置
    const typename Encoding::Ch* end = s + length;
    // 初始化计数
    SizeType count = 0;
    // 遍历字符串流
    while (is.src_ < end) {
        // 解码字符
        unsigned codepoint;
        if (!Encoding::Decode(is, &codepoint))
            return false;
        // 增加计数
        count++;
    }
    // 将计数结果写入输出指针
    *outCount = count;
    return true;
}

// 结束命名空间 rapidjson
} // namespace internal
// 结束命名空间 rapidjson
RAPIDJSON_NAMESPACE_END

// 结束条件编译
#endif // RAPIDJSON_INTERNAL_STRFUNC_H_
```