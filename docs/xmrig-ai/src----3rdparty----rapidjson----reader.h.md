# `xmrig\src\3rdparty\rapidjson\reader.h`

```
// 定义了 RAPIDJSON_READER_H_ 宏，用于避免重复包含
#ifndef RAPIDJSON_READER_H_
#define RAPIDJSON_READER_H_

// 包含相关头文件
#include "allocators.h"
#include "stream.h"
#include "encodedstream.h"
#include "internal/clzll.h"
#include "internal/meta.h"
#include "internal/stack.h"
#include "internal/strtod.h"
#include <limits>

// 根据条件包含特定的 SIMD 相关头文件
#if defined(RAPIDJSON_SIMD) && defined(_MSC_VER)
#include <intrin.h>
#pragma intrinsic(_BitScanForward)
#endif
#ifdef RAPIDJSON_SSE42
#include <nmmintrin.h>
#elif defined(RAPIDJSON_SSE2)
#include <emmintrin.h>
#elif defined(RAPIDJSON_NEON)
#include <arm_neon.h>
#endif

// 根据编译器类型设置特定的编译器指令
#ifdef __clang__
RAPIDJSON_DIAG_PUSH
RAPIDJSON_DIAG_OFF(old-style-cast)
RAPIDJSON_DIAG_OFF(padded)
RAPIDJSON_DIAG_OFF(switch-enum)
#elif defined(_MSC_VER)
RAPIDJSON_DIAG_PUSH
RAPIDJSON_DIAG_OFF(4127)  // conditional expression is constant
RAPIDJSON_DIAG_OFF(4702)  // unreachable code
#endif

#ifdef __GNUC__
RAPIDJSON_DIAG_PUSH
RAPIDJSON_DIAG_OFF(effc++)
#endif

// 定义了 RAPIDJSON_NOTHING 宏，用于表示空操作
//!@cond RAPIDJSON_HIDDEN_FROM_DOXYGEN
#define RAPIDJSON_NOTHING /* deliberately empty */
// 定义了 RAPIDJSON_PARSE_ERROR_EARLY_RETURN 宏，用于在解析过程中提前返回
#ifndef RAPIDJSON_PARSE_ERROR_EARLY_RETURN
#define RAPIDJSON_PARSE_ERROR_EARLY_RETURN(value) \
    RAPIDJSON_MULTILINEMACRO_BEGIN \
    if (RAPIDJSON_UNLIKELY(HasParseError())) { return value; } \
    RAPIDJSON_MULTILINEMACRO_END
#endif
// 定义了 RAPIDJSON_PARSE_ERROR_EARLY_RETURN_VOID 宏，用于在解析过程中提前返回空值
#define RAPIDJSON_PARSE_ERROR_EARLY_RETURN_VOID \
    # 如果遇到解析错误，立即返回RAPIDJSON_NOTHING
    RAPIDJSON_PARSE_ERROR_EARLY_RETURN(RAPIDJSON_NOTHING)
//!@endcond

/*! \def RAPIDJSON_PARSE_ERROR_NORETURN
    \ingroup RAPIDJSON_ERRORS
    \brief Macro to indicate a parse error.
    \param parseErrorCode \ref rapidjson::ParseErrorCode of the error
    \param offset  position of the error in JSON input (\c size_t)

    This macros can be used as a customization point for the internal
    error handling mechanism of RapidJSON.

    A common usage model is to throw an exception instead of requiring the
    caller to explicitly check the \ref rapidjson::GenericReader::Parse's
    return value:

    \code
    #define RAPIDJSON_PARSE_ERROR_NORETURN(parseErrorCode,offset) \
       throw ParseException(parseErrorCode, #parseErrorCode, offset)

    #include <stdexcept>               // std::runtime_error
    #include "rapidjson/error/error.h" // rapidjson::ParseResult

    struct ParseException : std::runtime_error, rapidjson::ParseResult {
      ParseException(rapidjson::ParseErrorCode code, const char* msg, size_t offset)
        : std::runtime_error(msg), ParseResult(code, offset) {}
    };

    #include "rapidjson/reader.h"
    \endcode

    \see RAPIDJSON_PARSE_ERROR, rapidjson::GenericReader::Parse
 */
#ifndef RAPIDJSON_PARSE_ERROR_NORETURN
#define RAPIDJSON_PARSE_ERROR_NORETURN(parseErrorCode, offset) \
    RAPIDJSON_MULTILINEMACRO_BEGIN \
    RAPIDJSON_ASSERT(!HasParseError()); /* Error can only be assigned once */ \
    SetParseError(parseErrorCode, offset); \
    RAPIDJSON_MULTILINEMACRO_END
#endif

/*! \def RAPIDJSON_PARSE_ERROR
    \ingroup RAPIDJSON_ERRORS
    \brief (Internal) macro to indicate and handle a parse error.
    \param parseErrorCode \ref rapidjson::ParseErrorCode of the error
    \param offset  position of the error in JSON input (\c size_t)

    Invokes RAPIDJSON_PARSE_ERROR_NORETURN and stops the parsing.

    \see RAPIDJSON_PARSE_ERROR_NORETURN
    \hideinitializer
 */
#ifndef RAPIDJSON_PARSE_ERROR
#define RAPIDJSON_PARSE_ERROR(parseErrorCode, offset) \
    RAPIDJSON_MULTILINEMACRO_BEGIN \
    # 使用宏定义RAPIDJSON_PARSE_ERROR_NORETURN，传入解析错误码和偏移量
    RAPIDJSON_PARSE_ERROR_NORETURN(parseErrorCode, offset); \
    # 使用宏定义RAPIDJSON_PARSE_ERROR_EARLY_RETURN_VOID，提前返回void类型
    RAPIDJSON_PARSE_ERROR_EARLY_RETURN_VOID; \
    # 结束多行宏定义
    RAPIDJSON_MULTILINEMACRO_END
#ifndef

#include "error/error.h" // 引入错误处理相关的头文件
#include "error/error.h" // ParseErrorCode, ParseResult

RAPIDJSON_NAMESPACE_BEGIN

///////////////////////////////////////////////////////////////////////////////
// ParseFlag

/*! \def RAPIDJSON_PARSE_DEFAULT_FLAGS
    \ingroup RAPIDJSON_CONFIG
    \brief User-defined kParseDefaultFlags definition.

    User can define this as any \c ParseFlag combinations.
*/
#ifndef RAPIDJSON_PARSE_DEFAULT_FLAGS
#define RAPIDJSON_PARSE_DEFAULT_FLAGS kParseNoFlags
#endif

//! Combination of parseFlags
/*! \see Reader::Parse, Document::Parse, Document::ParseInsitu, Document::ParseStream
 */
enum ParseFlag {
    kParseNoFlags = 0,              //!< 没有设置任何标志。
    kParseInsituFlag = 1,           //!< 原地（破坏性）解析。
    kParseValidateEncodingFlag = 2, //!< 验证 JSON 字符串的编码。
    kParseIterativeFlag = 4,        //!< 迭代（在函数调用栈大小方面具有恒定复杂度）解析。
    kParseStopWhenDoneFlag = 8,     //!< 从流中解析完整的 JSON 根后，停止进一步处理流的其余部分。当使用此标志时，解析器不会生成 kParseErrorDocumentRootNotSingular 错误。
    kParseFullPrecisionFlag = 16,   //!< 以完整精度解析数字（但速度较慢）。
    kParseCommentsFlag = 32,        //!< 允许单行（//）和多行（/**/）注释。
    kParseNumbersAsStringsFlag = 64,    //!< 将所有数字（整数/双精度数）解析为字符串。
    kParseTrailingCommasFlag = 128, //!< 允许对象和数组末尾的逗号。
    kParseNanAndInfFlag = 256,      //!< 允许解析 NaN、Inf、Infinity、-Inf 和 -Infinity 作为双精度数。
    kParseEscapedApostropheFlag = 512,  //!< 允许在字符串中转义撇号。
    kParseDefaultFlags = RAPIDJSON_PARSE_DEFAULT_FLAGS  //!< 默认解析标志。可以通过定义 RAPIDJSON_PARSE_DEFAULT_FLAGS 进行自定义
};

///////////////////////////////////////////////////////////////////////////////
// Handler

/*! \class rapidjson::Handler
    # 用于接收 GenericReader 解析后的事件的概念
    # 如果函数返回 true，则表示没有发生错误。如果返回 false，则表示事件发布者应该终止进程。
\code
// 定义 Handler 概念
concept Handler {
    typename Ch;

    // 检查是否为 null
    bool Null();
    // 处理布尔值
    bool Bool(bool b);
    // 处理整数
    bool Int(int i);
    // 处理无符号整数
    bool Uint(unsigned i);
    // 处理 64 位整数
    bool Int64(int64_t i);
    // 处理无符号 64 位整数
    bool Uint64(uint64_t i);
    // 处理双精度浮点数
    bool Double(double d);
    /// 通过 kParseNumbersAsStringsFlag 启用，字符串不以 null 结尾（使用长度）
    bool RawNumber(const Ch* str, SizeType length, bool copy);
    // 处理字符串
    bool String(const Ch* str, SizeType length, bool copy);
    // 开始对象
    bool StartObject();
    // 处理对象的键
    bool Key(const Ch* str, SizeType length, bool copy);
    // 结束对象
    bool EndObject(SizeType memberCount);
    // 开始数组
    bool StartArray();
    // 结束数组
    bool EndArray(SizeType elementCount);
};
\endcode
*/
///////////////////////////////////////////////////////////////////////////////
// BaseReaderHandler

//! 默认的 Handler 实现
/*! 这可以作为任何读取处理程序的基类。
    \note 实现了 Handler 概念
*/
template<typename Encoding = UTF8<>, typename Derived = void>
struct BaseReaderHandler {
    typedef typename Encoding::Ch Ch;

    typedef typename internal::SelectIf<internal::IsSame<Derived, void>, BaseReaderHandler, Derived>::Type Override;

    // 默认处理函数
    bool Default() { return true; }
    // 处理 null
    bool Null() { return static_cast<Override&>(*this).Default(); }
    // 处理布尔值
    bool Bool(bool) { return static_cast<Override&>(*this).Default(); }
    // 处理整数
    bool Int(int) { return static_cast<Override&>(*this).Default(); }
    // 处理无符号整数
    bool Uint(unsigned) { return static_cast<Override&>(*this).Default(); }
    // 处理 64 位整数
    bool Int64(int64_t) { return static_cast<Override&>(*this).Default(); }
    // 处理无符号 64 位整数
    bool Uint64(uint64_t) { return static_cast<Override&>(*this).Default(); }
    // 处理双精度浮点数
    bool Double(double) { return static_cast<Override&>(*this).Default(); }
    /// 通过 kParseNumbersAsStringsFlag 启用，字符串不以 null 结尾（使用长度）
    bool RawNumber(const Ch* str, SizeType len, bool copy) { return static_cast<Override&>(*this).String(str, len, copy); }
    // 处理字符串
    bool String(const Ch*, SizeType, bool) { return static_cast<Override&>(*this).Default(); }
    # 开始解析 JSON 对象
    bool StartObject() { return static_cast<Override&>(*this).Default(); }
    # 解析 JSON 键值对的键
    bool Key(const Ch* str, SizeType len, bool copy) { return static_cast<Override&>(*this).String(str, len, copy); }
    # 结束解析 JSON 对象
    bool EndObject(SizeType) { return static_cast<Override&>(*this).Default(); }
    # 开始解析 JSON 数组
    bool StartArray() { return static_cast<Override&>(*this).Default(); }
    # 结束解析 JSON 数组
    bool EndArray(SizeType) { return static_cast<Override&>(*this).Default(); }
};

///////////////////////////////////////////////////////////////////////////////
// StreamLocalCopy

namespace internal {

template<typename Stream, int = StreamTraits<Stream>::copyOptimization>
class StreamLocalCopy;

//! Do copy optimization.
template<typename Stream>
class StreamLocalCopy<Stream, 1> {
public:
    // 构造函数，保存原始流的引用
    StreamLocalCopy(Stream& original) : s(original), original_(original) {}
    // 析构函数，将保存的流重新赋值给原始流
    ~StreamLocalCopy() { original_ = s; }

    Stream s;

private:
    // 禁用赋值操作符
    StreamLocalCopy& operator=(const StreamLocalCopy&) /* = delete */;

    Stream& original_;
};

//! Keep reference.
template<typename Stream>
class StreamLocalCopy<Stream, 0> {
public:
    // 构造函数，保存原始流的引用
    StreamLocalCopy(Stream& original) : s(original) {}

    Stream& s;

private:
    // 禁用赋值操作符
    StreamLocalCopy& operator=(const StreamLocalCopy&) /* = delete */;
};

} // namespace internal

///////////////////////////////////////////////////////////////////////////////
// SkipWhitespace

//! Skip the JSON white spaces in a stream.
/*! \param is A input stream for skipping white spaces.
    \note This function has SSE2/SSE4.2 specialization.
*/
template<typename InputStream>
void SkipWhitespace(InputStream& is) {
    // 创建流的本地拷贝
    internal::StreamLocalCopy<InputStream> copy(is);
    // 获取本地拷贝的引用
    InputStream& s(copy.s);

    typename InputStream::Ch c;
    // 跳过空白字符
    while ((c = s.Peek()) == ' ' || c == '\n' || c == '\r' || c == '\t')
        s.Take();
}

// 跳过空白字符，返回指向非空白字符的指针
inline const char* SkipWhitespace(const char* p, const char* end) {
    while (p != end && (*p == ' ' || *p == '\n' || *p == '\r' || *p == '\t'))
        ++p;
    return p;
}

#ifdef RAPIDJSON_SSE42
//! Skip whitespace with SSE 4.2 pcmpistrm instruction, testing 16 8-byte characters at once.
inline const char *SkipWhitespace_SIMD(const char* p) {
    // 如果当前字符是空白字符，则向前移动一个字符
    if (*p == ' ' || *p == '\n' || *p == '\r' || *p == '\t')
        ++p;
    else
        return p;

    // 16字节对齐到下一个边界
    // 将指针 p 转换为 size_t 类型，加上 15 后按位与 15 的补码，再转换回 const char* 类型，使其指向下一个 16 字节对齐的地址
    const char* nextAligned = reinterpret_cast<const char*>((reinterpret_cast<size_t>(p) + 15) & static_cast<size_t>(~15));
    // 循环直到 p 指向下一个 16 字节对齐的地址
    while (p != nextAligned)
        // 如果当前字符是空格、换行符、回车符或制表符，则指针向后移动一位
        if (*p == ' ' || *p == '\n' || *p == '\r' || *p == '\t')
            ++p;
        else
            // 如果当前字符不是空白字符，则返回当前指针位置
            return p;

    // 使用 SIMD 处理字符串的剩余部分
    // 定义包含空白字符的静态字符数组
    static const char whitespace[16] = " \n\r\t";
    // 加载空白字符数组到 SIMD 寄存器
    const __m128i w = _mm_loadu_si128(reinterpret_cast<const __m128i *>(&whitespace[0]));

    // 无限循环，每次移动 16 个字节
    for (;; p += 16) {
        // 加载 16 个字符到 SIMD 寄存器
        const __m128i s = _mm_load_si128(reinterpret_cast<const __m128i *>(p));
        // 比较 s 和 w 寄存器中的字符，找到第一个不是空白字符的位置
        const int r = _mm_cmpistri(w, s, _SIDD_UBYTE_OPS | _SIDD_CMP_EQUAL_ANY | _SIDD_LEAST_SIGNIFICANT | _SIDD_NEGATIVE_POLARITY);
        // 如果找到不是空白字符的位置，则返回当前指针位置加上 r
        if (r != 16)    // some of characters is non-whitespace
            return p + r;
    }
// 跳过输入字符串中的空白字符，使用 SIMD 指令集
inline const char *SkipWhitespace_SIMD(const char* p, const char* end) {
    // 如果当前字符不是空白字符，则直接返回下一个字符的指针
    if (p != end && (*p == ' ' || *p == '\n' || *p == '\r' || *p == '\t'))
        ++p;
    else
        return p;

    // 使用 SIMD 指令集处理字符串中间部分
    static const char whitespace[16] = " \n\r\t";
    const __m128i w = _mm_loadu_si128(reinterpret_cast<const __m128i *>(&whitespace[0]));

    // 使用 SIMD 指令集循环处理字符串
    for (; p <= end - 16; p += 16) {
        const __m128i s = _mm_loadu_si128(reinterpret_cast<const __m128i *>(p));
        const int r = _mm_cmpistri(w, s, _SIDD_UBYTE_OPS | _SIDD_CMP_EQUAL_ANY | _SIDD_LEAST_SIGNIFICANT | _SIDD_NEGATIVE_POLARITY);
        if (r != 16)    // 如果存在非空白字符，则返回该字符的指针
            return p + r;
    }

    // 如果字符串中间部分没有找到非空白字符，则调用 SkipWhitespace 函数处理剩余部分
    return SkipWhitespace(p, end);
}

#elif defined(RAPIDJSON_SSE2)

//! 使用 SSE2 指令集跳过空白字符，一次处理 16 个 8 字节字符
inline const char *SkipWhitespace_SIMD(const char* p) {
    // 如果当前字符不是空白字符，则直接返回下一个字符的指针
    if (*p == ' ' || *p == '\n' || *p == '\r' || *p == '\t')
        ++p;
    else
        return p;

    // 将指针向下对齐到 16 字节边界
    const char* nextAligned = reinterpret_cast<const char*>((reinterpret_cast<size_t>(p) + 15) & static_cast<size_t>(~15));
    // 循环处理直到指针对齐到 16 字节边界
    while (p != nextAligned)
        if (*p == ' ' || *p == '\n' || *p == '\r' || *p == '\t')
            ++p;
        else
            return p;

    // 处理字符串剩余部分
    #define C16(c) { c, c, c, c, c, c, c, c, c, c, c, c, c, c, c, c }
    static const char whitespaces[4][16] = { C16(' '), C16('\n'), C16('\r'), C16('\t') };
    #undef C16

    const __m128i w0 = _mm_loadu_si128(reinterpret_cast<const __m128i *>(&whitespaces[0][0]));
    const __m128i w1 = _mm_loadu_si128(reinterpret_cast<const __m128i *>(&whitespaces[1][0]));
    const __m128i w2 = _mm_loadu_si128(reinterpret_cast<const __m128i *>(&whitespaces[2][0]));
    // 从whitespaces数组中加载第四个元素的值到__m128i类型的变量w3
    const __m128i w3 = _mm_loadu_si128(reinterpret_cast<const __m128i *>(&whitespaces[3][0]));

    // 无限循环，每次移动16个字节
    for (;; p += 16) {
        // 从指针p处加载16个字节到__m128i类型的变量s
        const __m128i s = _mm_load_si128(reinterpret_cast<const __m128i *>(p));
        // 比较s中的每个字节是否等于w0中的对应字节，将结果存储到x中
        __m128i x = _mm_cmpeq_epi8(s, w0);
        // 将s中的每个字节与w1中的对应字节进行比较，将结果存储到x中
        x = _mm_or_si128(x, _mm_cmpeq_epi8(s, w1));
        // 将s中的每个字节与w2中的对应字节进行比较，将结果存储到x中
        x = _mm_or_si128(x, _mm_cmpeq_epi8(s, w2));
        // 将s中的每个字节与w3中的对应字节进行比较，将结果存储到x中
        x = _mm_or_si128(x, _mm_cmpeq_epi8(s, w3));
        // 将x中每个字节的比较结果取反，转换为16位无符号整数
        unsigned short r = static_cast<unsigned short>(~_mm_movemask_epi8(x));
        // 如果r不等于0，说明有非空白字符
        if (r != 0) {   // some of characters may be non-whitespace
#ifdef _MSC_VER         // 如果是在 MSVC 编译器下
            unsigned long offset;  // 定义一个无符号长整型变量 offset
            _BitScanForward(&offset, r);  // 使用 _BitScanForward 函数找到第一个非空白字符的索引
            return p + offset;  // 返回 p 加上偏移量的指针
#else
            return p + __builtin_ffs(r) - 1;  // 否则返回 p 加上 __builtin_ffs(r) 函数返回值减 1 的指针
#endif
        }
    }
}

inline const char *SkipWhitespace_SIMD(const char* p, const char* end) {  // 定义一个跳过空白字符的 SIMD 函数，参数为起始指针和结束指针
    // 如果起始指针不等于结束指针，并且当前字符是空格、换行、回车或制表符，则快速返回
    if (p != end && (*p == ' ' || *p == '\n' || *p == '\r' || *p == '\t'))
        ++p;  // 指针向后移动一位
    else
        return p;  // 否则直接返回当前指针

    // 定义一个宏，用于初始化 16 个相同字符的数组
    #define C16(c) { c, c, c, c, c, c, c, c, c, c, c, c, c, c, c, c }
    static const char whitespaces[4][16] = { C16(' '), C16('\n'), C16('\r'), C16('\t') };  // 定义一个包含空白字符的静态数组
    #undef C16  // 取消宏定义

    const __m128i w0 = _mm_loadu_si128(reinterpret_cast<const __m128i *>(&whitespaces[0][0]));  // 使用 SIMD 加载第一个空白字符数组
    const __m128i w1 = _mm_loadu_si128(reinterpret_cast<const __m128i *>(&whitespaces[1][0]));  // 使用 SIMD 加载第二个空白字符数组
    const __m128i w2 = _mm_loadu_si128(reinterpret_cast<const __m128i *>(&whitespaces[2][0]));  // 使用 SIMD 加载第三个空白字符数组
    const __m128i w3 = _mm_loadu_si128(reinterpret_cast<const __m128i *>(&whitespaces[3][0]));  // 使用 SIMD 加载第四个空白字符数组

    for (; p <= end - 16; p += 16) {  // 循环遍历字符串，每次移动 16 个字符
        const __m128i s = _mm_loadu_si128(reinterpret_cast<const __m128i *>(p));  // 使用 SIMD 加载 16 个字符
        __m128i x = _mm_cmpeq_epi8(s, w0);  // 使用 SIMD 比较是否相等
        x = _mm_or_si128(x, _mm_cmpeq_epi8(s, w1));  // 使用 SIMD 或运算
        x = _mm_or_si128(x, _mm_cmpeq_epi8(s, w2));  // 使用 SIMD 或运算
        x = _mm_or_si128(x, _mm_cmpeq_epi8(s, w3));  // 使用 SIMD 或运算
        unsigned short r = static_cast<unsigned short>(~_mm_movemask_epi8(x));  // 将结果转换为无符号短整型
        if (r != 0) {   // 如果存在非空白字符
#ifdef _MSC_VER         // 如果是在 MSVC 编译器下
            unsigned long offset;  // 定义一个无符号长整型变量 offset
            _BitScanForward(&offset, r);  // 使用 _BitScanForward 函数找到第一个非空白字符的索引
            return p + offset;  // 返回 p 加上偏移量的指针
#else
            return p + __builtin_ffs(r) - 1;  // 否则返回 p 加上 __builtin_ffs(r) 函数返回值减 1 的指针
#endif
        }
    }

    return SkipWhitespace(p, end);  // 返回调用 SkipWhitespace 函数的结果
}

#elif defined(RAPIDJSON_NEON)

//! 使用 ARM Neon 指令跳过空白字符，一次测试 16 个 8 字节字符。
inline const char *SkipWhitespace_SIMD(const char* p) {  // 定义一个使用 SIMD 跳过空白字符的函数，参数为起始指针
    // 如果指针指向的字符是空格、换行、回车或制表符，则向前移动一个字符
    if (*p == ' ' || *p == '\n' || *p == '\r' || *p == '\t')
        ++p;
    else
        return p;

    // 将指针移动到下一个16字节边界对齐位置
    const char* nextAligned = reinterpret_cast<const char*>((reinterpret_cast<size_t>(p) + 15) & static_cast<size_t>(~15));
    while (p != nextAligned)
        if (*p == ' ' || *p == '\n' || *p == '\r' || *p == '\t')
            ++p;
        else
            return p;

    // 创建包含空格、换行、回车和制表符的16字节数据块
    const uint8x16_t w0 = vmovq_n_u8(' ');
    const uint8x16_t w1 = vmovq_n_u8('\n');
    const uint8x16_t w2 = vmovq_n_u8('\r');
    const uint8x16_t w3 = vmovq_n_u8('\t');

    // 循环处理每个16字节数据块
    for (;; p += 16) {
        const uint8x16_t s = vld1q_u8(reinterpret_cast<const uint8_t *>(p));  // 加载16字节数据块
        uint8x16_t x = vceqq_u8(s, w0);  // 比较是否包含空格
        x = vorrq_u8(x, vceqq_u8(s, w1));  // 比较是否包含换行
        x = vorrq_u8(x, vceqq_u8(s, w2));  // 比较是否包含回车
        x = vorrq_u8(x, vceqq_u8(s, w3));  // 比较是否包含制表符

        x = vmvnq_u8(x);  // 取反
        x = vrev64q_u8(x);  // 反转64位
        uint64_t low = vgetq_lane_u64(vreinterpretq_u64_u8(x), 0);  // 提取低64位
        uint64_t high = vgetq_lane_u64(vreinterpretq_u64_u8(x), 1);  // 提取高64位

        if (low == 0) {
            if (high != 0) {
                uint32_t lz = internal::clzll(high);  // 计算高64位的前导零
                return p + 8 + (lz >> 3);  // 返回指向第一个非空白字符的指针
            }
        } else {
            uint32_t lz = internal::clzll(low);  // 计算低64位的前导零
            return p + (lz >> 3);  // 返回指向第一个非空白字符的指针
        }
    }
// 跳过输入字符串中的空白字符，使用 SIMD 指令集加速
inline const char *SkipWhitespace_SIMD(const char* p, const char* end) {
    // 如果当前字符不是空白字符，则直接返回下一个字符的指针
    if (p != end && (*p == ' ' || *p == '\n' || *p == '\r' || *p == '\t'))
        ++p;
    else
        return p;

    // 使用 SIMD 加载空白字符
    const uint8x16_t w0 = vmovq_n_u8(' ');
    const uint8x16_t w1 = vmovq_n_u8('\n');
    const uint8x16_t w2 = vmovq_n_u8('\r');
    const uint8x16_t w3 = vmovq_n_u8('\t');

    // 循环处理输入字符串，每次处理 16 个字符
    for (; p <= end - 16; p += 16) {
        // 使用 SIMD 指令进行比较，找出空白字符
        const uint8x16_t s = vld1q_u8(reinterpret_cast<const uint8_t *>(p));
        uint8x16_t x = vceqq_u8(s, w0);
        x = vorrq_u8(x, vceqq_u8(s, w1));
        x = vorrq_u8(x, vceqq_u8(s, w2));
        x = vorrq_u8(x, vceqq_u8(s, w3));

        x = vmvnq_u8(x);                       // 取反
        x = vrev64q_u8(x);                     // 64 位反转
        uint64_t low = vgetq_lane_u64(vreinterpretq_u64_u8(x), 0);   // 提取低位
        uint64_t high = vgetq_lane_u64(vreinterpretq_u64_u8(x), 1);  // 提取高位

        // 根据空白字符的位置返回相应的指针
        if (low == 0) {
            if (high != 0) {
                uint32_t lz = internal::clzll(high);
                return p + 8 + (lz >> 3);
            }
        } else {
            uint32_t lz = internal::clzll(low);
            return p + (lz >> 3);
        }
    }

    // 如果输入字符串长度不足 16 个字符，则调用非 SIMD 版本的跳过空白字符函数
    return SkipWhitespace(p, end);
}

#endif // RAPIDJSON_NEON

#ifdef RAPIDJSON_SIMD
// 为 InsituStringStream 的跳过空白字符函数进行模板函数特化
template<> inline void SkipWhitespace(InsituStringStream& is) {
    is.src_ = const_cast<char*>(SkipWhitespace_SIMD(is.src_));
}

// 为 StringStream 的跳过空白字符函数进行模板函数特化
template<> inline void SkipWhitespace(StringStream& is) {
    is.src_ = SkipWhitespace_SIMD(is.src_);
}

// 为 EncodedInputStream<UTF8<>, MemoryStream> 的跳过空白字符函数进行模板函数特化
template<> inline void SkipWhitespace(EncodedInputStream<UTF8<>, MemoryStream>& is) {
    is.is_.src_ = SkipWhitespace_SIMD(is.is_.src_, is.is_.end_);
}
#endif // RAPIDJSON_SIMD

///////////////////////////////////////////////////////////////////////////////
// GenericReader
//! SAX-style JSON parser. Use \ref Reader for UTF8 encoding and default allocator.
/*! GenericReader parses JSON text from a stream, and send events synchronously to an
    object implementing Handler concept.

    It needs to allocate a stack for storing a single decoded string during
    non-destructive parsing.

    For in-situ parsing, the decoded string is directly written to the source
    text string, no temporary buffer is required.

    A GenericReader object can be reused for parsing multiple JSON text.

    \tparam SourceEncoding Encoding of the input stream.
    \tparam TargetEncoding Encoding of the parse output.
    \tparam StackAllocator Allocator type for stack.
*/
template <typename SourceEncoding, typename TargetEncoding, typename StackAllocator = CrtAllocator>
class GenericReader {
public:
    typedef typename SourceEncoding::Ch Ch; //!< SourceEncoding character type

    //! Constructor.
    /*! \param stackAllocator Optional allocator for allocating stack memory. (Only use for non-destructive parsing)
        \param stackCapacity stack capacity in bytes for storing a single decoded string.  (Only use for non-destructive parsing)
    */
    GenericReader(StackAllocator* stackAllocator = 0, size_t stackCapacity = kDefaultStackCapacity) :
        stack_(stackAllocator, stackCapacity), parseResult_(), state_(IterativeParsingStartState) {}

    //! Parse JSON text.
    /*! \tparam parseFlags Combination of \ref ParseFlag.
        \tparam InputStream Type of input stream, implementing Stream concept.
        \tparam Handler Type of handler, implementing Handler concept.
        \param is Input stream to be parsed.
        \param handler The handler to receive events.
        \return Whether the parsing is successful.
    */
    template <unsigned parseFlags, typename InputStream, typename Handler>
    // 解析 JSON 文本，根据解析标志选择迭代解析或普通解析
    ParseResult Parse(InputStream& is, Handler& handler) {
        // 如果解析标志包含迭代解析标志，则调用迭代解析函数
        if (parseFlags & kParseIterativeFlag)
            return IterativeParse<parseFlags>(is, handler);

        // 清空解析结果
        parseResult_.Clear();

        // 在退出时清空堆栈
        ClearStackOnExit scope(*this);

        // 跳过空白字符和注释
        SkipWhitespaceAndComments<parseFlags>(is);
        RAPIDJSON_PARSE_ERROR_EARLY_RETURN(parseResult_);

        // 如果输入流为空，则返回解析错误
        if (RAPIDJSON_UNLIKELY(is.Peek() == '\0')) {
            RAPIDJSON_PARSE_ERROR_NORETURN(kParseErrorDocumentEmpty, is.Tell());
            RAPIDJSON_PARSE_ERROR_EARLY_RETURN(parseResult_);
        }
        else {
            // 解析值
            ParseValue<parseFlags>(is, handler);
            RAPIDJSON_PARSE_ERROR_EARLY_RETURN(parseResult_);

            // 如果解析标志不包含解析完成后停止标志，则继续解析
            if (!(parseFlags & kParseStopWhenDoneFlag)) {
                // 跳过空白字符和注释
                SkipWhitespaceAndComments<parseFlags>(is);
                RAPIDJSON_PARSE_ERROR_EARLY_RETURN(parseResult_);

                // 如果输入流不为空，则返回解析错误
                if (RAPIDJSON_UNLIKELY(is.Peek() != '\0')) {
                    RAPIDJSON_PARSE_ERROR_NORETURN(kParseErrorDocumentRootNotSingular, is.Tell());
                    RAPIDJSON_PARSE_ERROR_EARLY_RETURN(parseResult_);
                }
            }
        }

        return parseResult_;
    }

    //! 解析 JSON 文本（使用 \ref kParseDefaultFlags）
    /*! \tparam InputStream 输入流的类型，实现了流概念
        \tparam Handler 处理程序的类型，实现了处理程序概念
        \param is 要解析的输入流
        \param handler 用于接收事件的处理程序
        \return 解析是否成功
    */
    template <typename InputStream, typename Handler>
    ParseResult Parse(InputStream& is, Handler& handler) {
        return Parse<kParseDefaultFlags>(is, handler);
    }

    //! 初始化逐标记解析 JSON 文本
    /*!
     */
    void IterativeParseInit() {
        // 清空解析结果
        parseResult_.Clear();
        // 设置状态为迭代解析的起始状态
        state_ = IterativeParsingStartState;
    }

    //! 从 JSON 文本中解析一个标记
    /*! 
        \tparam InputStream Type of input stream, implementing Stream concept
        \tparam Handler Type of handler, implementing Handler concept.
        \param is Input stream to be parsed.
        \param handler The handler to receive events.
        \return Whether the parsing is successful.
     */
    template <unsigned parseFlags, typename InputStream, typename Handler>
    // 使用迭代方式解析输入流中的 JSON 数据，直到遇到结束符或错误
    bool IterativeParseNext(InputStream& is, Handler& handler) {
        // 当输入流中下一个字符不是结束符时循环执行
        while (RAPIDJSON_LIKELY(is.Peek() != '\0')) {
            // 跳过空白字符和注释
            SkipWhitespaceAndComments<parseFlags>(is);

            // 将下一个字符标记为一个 Token
            Token t = Tokenize(is.Peek());
            // 预测下一个状态
            IterativeParsingState n = Predict(state_, t);
            // 根据当前状态、下一个 Token、下一个状态和输入流执行状态转移
            IterativeParsingState d = Transit<parseFlags>(state_, t, n, is, handler);

            // 如果已经完成或者遇到错误...
            if (RAPIDJSON_UNLIKELY(IsIterativeParsingCompleteState(d))) {
                // 报告错误
                if (d == IterativeParsingErrorState) {
                    HandleError(state_, is);
                    return false;
                }

                // 转换到完成状态
                RAPIDJSON_ASSERT(d == IterativeParsingFinishState);
                state_ = d;

                // 如果未设置 StopWhenDone 标志...
                if (!(parseFlags & kParseStopWhenDoneFlag)) {
                    // ...并且找到额外的非空白数据...
                    SkipWhitespaceAndComments<parseFlags>(is);
                    if (is.Peek() != '\0') {
                        // ...则视为错误
                        HandleError(state_, is);
                        return false;
                    }
                }

                // 成功！解析完成！
                return true;
            }

            // 转换到新状态
            state_ = d;

            // 如果解析到的不是分隔符，表示已经调用了处理程序，现在可以返回 true
            if (!IsIterativeParsingDelimiterState(n))
                return true;
        }

        // 已经到达文件末尾
        stack_.Clear();

        // 如果状态不是完成状态，则报告错误
        if (state_ != IterativeParsingFinishState) {
            HandleError(state_, is);
            return false;
        }

        return true;
    }

    //! 检查逐个标记解析 JSON 文本是否完成
    # 返回 JSON 是否已完全解码的布尔值
    RAPIDJSON_FORCEINLINE bool IterativeParseComplete() const {
        return IsIterativeParsingCompleteState(state_);
    }
    
    # 检查上次解析是否发生了解析错误
    bool HasParseError() const { return parseResult_.IsError(); }
    
    # 获取上次解析的解析错误代码
    ParseErrorCode GetParseErrorCode() const { return parseResult_.Code(); }
    
    # 获取上次解析错误的输入位置，如果没有错误则返回0
    size_t GetErrorOffset() const { return parseResult_.Offset(); }
protected:
    // 设置解析错误代码和偏移量
    void SetParseError(ParseErrorCode code, size_t offset) { parseResult_.Set(code, offset); }

private:
    // 禁止复制构造函数和赋值运算符
    GenericReader(const GenericReader&);
    GenericReader& operator=(const GenericReader&);

    // 清空堆栈
    void ClearStack() { stack_.Clear(); }

    // 在任何退出 ParseStream 的情况下清空堆栈，例如由于异常
    struct ClearStackOnExit {
        explicit ClearStackOnExit(GenericReader& r) : r_(r) {}
        ~ClearStackOnExit() { r_.ClearStack(); }
    private:
        GenericReader& r_;
        ClearStackOnExit(const ClearStackOnExit&);
        ClearStackOnExit& operator=(const ClearStackOnExit&);
    };

    // 跳过空白和注释
    template<unsigned parseFlags, typename InputStream>
    void SkipWhitespaceAndComments(InputStream& is) {
        SkipWhitespace(is);

        if (parseFlags & kParseCommentsFlag) {
            while (RAPIDJSON_UNLIKELY(Consume(is, '/'))) {
                if (Consume(is, '*')) {
                    while (true) {
                        if (RAPIDJSON_UNLIKELY(is.Peek() == '\0'))
                            RAPIDJSON_PARSE_ERROR(kParseErrorUnspecificSyntaxError, is.Tell());
                        else if (Consume(is, '*')) {
                            if (Consume(is, '/'))
                                break;
                        }
                        else
                            is.Take();
                    }
                }
                else if (RAPIDJSON_LIKELY(Consume(is, '/')))
                    while (is.Peek() != '\0' && is.Take() != '\n') {}
                else
                    RAPIDJSON_PARSE_ERROR(kParseErrorUnspecificSyntaxError, is.Tell());

                SkipWhitespace(is);
            }
        }
    }

    // 解析对象：{ string : value, ... }
    template<unsigned parseFlags, typename InputStream, typename Handler>
    }

    // 解析数组：[ value, ... ]
    template<unsigned parseFlags, typename InputStream, typename Handler>
    // 解析 JSON 数组
    void ParseArray(InputStream& is, Handler& handler) {
        // 断言下一个字符是 '['
        RAPIDJSON_ASSERT(is.Peek() == '[');
        // 跳过 '['
        is.Take();

        // 如果 handler.StartArray() 返回 false，则抛出解析错误
        if (RAPIDJSON_UNLIKELY(!handler.StartArray()))
            RAPIDJSON_PARSE_ERROR(kParseErrorTermination, is.Tell());

        // 跳过空白字符和注释
        SkipWhitespaceAndComments<parseFlags>(is);
        RAPIDJSON_PARSE_ERROR_EARLY_RETURN_VOID;

        // 如果遇到空数组，则调用 handler.EndArray(0) 并返回
        if (Consume(is, ']')) {
            if (RAPIDJSON_UNLIKELY(!handler.EndArray(0))) // empty array
                RAPIDJSON_PARSE_ERROR(kParseErrorTermination, is.Tell());
            return;
        }

        // 循环解析数组元素
        for (SizeType elementCount = 0;;) {
            // 解析数组元素的值
            ParseValue<parseFlags>(is, handler);
            RAPIDJSON_PARSE_ERROR_EARLY_RETURN_VOID;

            // 增加元素计数，并跳过空白字符和注释
            ++elementCount;
            SkipWhitespaceAndComments<parseFlags>(is);
            RAPIDJSON_PARSE_ERROR_EARLY_RETURN_VOID;

            // 如果遇到逗号，则继续解析下一个元素
            if (Consume(is, ',')) {
                SkipWhitespaceAndComments<parseFlags>(is);
                RAPIDJSON_PARSE_ERROR_EARLY_RETURN_VOID;
            }
            // 如果遇到 ']'，则调用 handler.EndArray(elementCount) 并返回
            else if (Consume(is, ']')) {
                if (RAPIDJSON_UNLIKELY(!handler.EndArray(elementCount)))
                    RAPIDJSON_PARSE_ERROR(kParseErrorTermination, is.Tell());
                return;
            }
            // 如果既不是逗号也不是 ']'，则抛出解析错误
            else
                RAPIDJSON_PARSE_ERROR(kParseErrorArrayMissCommaOrSquareBracket, is.Tell());

            // 如果解析标志包含 kParseTrailingCommasFlag，并且下一个字符是 ']'，则调用 handler.EndArray(elementCount) 并返回
            if (parseFlags & kParseTrailingCommasFlag) {
                if (is.Peek() == ']') {
                    if (RAPIDJSON_UNLIKELY(!handler.EndArray(elementCount)))
                        RAPIDJSON_PARSE_ERROR(kParseErrorTermination, is.Tell());
                    is.Take();
                    return;
                }
            }
        }
    }

    template<unsigned parseFlags, typename InputStream, typename Handler>
    // 解析 null 值
    void ParseNull(InputStream& is, Handler& handler) {
        // 断言下一个字符是 'n'
        RAPIDJSON_ASSERT(is.Peek() == 'n');
        // 读取并移除下一个字符
        is.Take();

        // 如果下一个字符依次是 'u', 'l', 'l'，则表示解析成功，调用 handler 处理 null 值
        if (RAPIDJSON_LIKELY(Consume(is, 'u') && Consume(is, 'l') && Consume(is, 'l'))) {
            if (RAPIDJSON_UNLIKELY(!handler.Null()))
                // 如果 handler 处理失败，报告解析错误
                RAPIDJSON_PARSE_ERROR(kParseErrorTermination, is.Tell());
        }
        else
            // 如果不是期望的字符序列，报告值无效的解析错误
            RAPIDJSON_PARSE_ERROR(kParseErrorValueInvalid, is.Tell());
    }

    // 解析 true 值
    template<unsigned parseFlags, typename InputStream, typename Handler>
    void ParseTrue(InputStream& is, Handler& handler) {
        // 断言下一个字符是 't'
        RAPIDJSON_ASSERT(is.Peek() == 't');
        // 读取并移除下一个字符
        is.Take();

        // 如果下一个字符依次是 'r', 'u', 'e'，则表示解析成功，调用 handler 处理 true 值
        if (RAPIDJSON_LIKELY(Consume(is, 'r') && Consume(is, 'u') && Consume(is, 'e'))) {
            if (RAPIDJSON_UNLIKELY(!handler.Bool(true)))
                // 如果 handler 处理失败，报告解析错误
                RAPIDJSON_PARSE_ERROR(kParseErrorTermination, is.Tell());
        }
        else
            // 如果不是期望的字符序列，报告值无效的解析错误
            RAPIDJSON_PARSE_ERROR(kParseErrorValueInvalid, is.Tell());
    }

    // 解析 false 值
    template<unsigned parseFlags, typename InputStream, typename Handler>
    void ParseFalse(InputStream& is, Handler& handler) {
        // 断言下一个字符是 'f'
        RAPIDJSON_ASSERT(is.Peek() == 'f');
        // 读取并移除下一个字符
        is.Take();

        // 如果下一个字符依次是 'a', 'l', 's', 'e'，则表示解析成功，调用 handler 处理 false 值
        if (RAPIDJSON_LIKELY(Consume(is, 'a') && Consume(is, 'l') && Consume(is, 's') && Consume(is, 'e'))) {
            if (RAPIDJSON_UNLIKELY(!handler.Bool(false)))
                // 如果 handler 处理失败，报告解析错误
                RAPIDJSON_PARSE_ERROR(kParseErrorTermination, is.Tell());
        }
        else
            // 如果不是期望的字符序列，报告值无效的解析错误
            RAPIDJSON_PARSE_ERROR(kParseErrorValueInvalid, is.Tell());
    }

    // 从输入流中消耗一个字符，如果和期望的字符相同则返回 true，否则返回 false
    template<typename InputStream>
    RAPIDJSON_FORCEINLINE static bool Consume(InputStream& is, typename InputStream::Ch expect) {
        if (RAPIDJSON_LIKELY(is.Peek() == expect)) {
            is.Take();
            return true;
        }
        else
            return false;
    }

    // 在 ParseString() 中解析 \uXXXX 形式的四位十六进制数的辅助函数
    template<typename InputStream>
    // 从输入流中解析16进制数，返回解析出的无符号整数
    unsigned ParseHex4(InputStream& is, size_t escapeOffset) {
        // 初始化 codepoint 为 0
        unsigned codepoint = 0;
        // 循环4次，解析4个16进制数字
        for (int i = 0; i < 4; i++) {
            // 读取输入流的下一个字符
            Ch c = is.Peek();
            // 将 codepoint 左移4位
            codepoint <<= 4;
            // 将当前字符转换为无符号整数并加到 codepoint 上
            codepoint += static_cast<unsigned>(c);
            // 根据字符的范围，将 codepoint 转换为对应的数字
            if (c >= '0' && c <= '9')
                codepoint -= '0';
            else if (c >= 'A' && c <= 'F')
                codepoint -= 'A' - 10;
            else if (c >= 'a' && c <= 'f')
                codepoint -= 'a' - 10;
            else {
                // 如果字符不在16进制范围内，抛出解析错误
                RAPIDJSON_PARSE_ERROR_NORETURN(kParseErrorStringUnicodeEscapeInvalidHex, escapeOffset);
                RAPIDJSON_PARSE_ERROR_EARLY_RETURN(0);
            }
            // 从输入流中取出一个字符
            is.Take();
        }
        // 返回解析出的无符号整数
        return codepoint;
    }

    template <typename CharType>
    class StackStream {
    public:
        typedef CharType Ch;

        // 构造函数，初始化栈和长度
        StackStream(internal::Stack<StackAllocator>& stack) : stack_(stack), length_(0) {}
        // 将字符放入栈中
        RAPIDJSON_FORCEINLINE void Put(Ch c) {
            *stack_.template Push<Ch>() = c;
            ++length_;
        }

        // 将指定数量的字符压入栈中
        RAPIDJSON_FORCEINLINE void* Push(SizeType count) {
            length_ += count;
            return stack_.template Push<Ch>(count);
        }

        // 返回当前长度
        size_t Length() const { return length_; }

        // 从栈中弹出字符
        Ch* Pop() {
            return stack_.template Pop<Ch>(length_);
        }

    private:
        // 禁止拷贝和赋值
        StackStream(const StackStream&);
        StackStream& operator=(const StackStream&);

        // 栈和长度
        internal::Stack<StackAllocator>& stack_;
        SizeType length_;
    };

    // 解析字符串并生成字符串事件。对于 kParseInsituFlag 有不同的代码路径。
    template<unsigned parseFlags, typename InputStream, typename Handler>
    // 解析字符串并将结果输出到流中
    // 此函数处理前缀/后缀双引号，转义和可选的编码验证
    template<unsigned parseFlags, typename SEncoding, typename TEncoding, typename InputStream, typename OutputStream>
    RAPIDJSON_FORCEINLINE void ParseStringToStream(InputStream& is, OutputStream& os) {
//!@cond RAPIDJSON_HIDDEN_FROM_DOXYGEN
// 定义一个名为 Z16 的宏，用于初始化 escape 数组
#define Z16 0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0
        // 初始化 escape 数组，用于存储字符的转义信息
        static const char escape[256] = {
            Z16, Z16, 0, 0,'\"', 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, '/',
            Z16, Z16, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,'\\', 0, 0, 0,
            0, 0,'\b', 0, 0, 0,'\f', 0, 0, 0, 0, 0, 0, 0,'\n', 0,
            0, 0,'\r', 0,'\t', 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
            Z16, Z16, Z16, Z16, Z16, Z16, Z16, Z16
        };
// 取消宏定义 Z16
#undef Z16
    }

    // 定义一个模板函数，用于扫描和复制未转义的字符串
    template<typename InputStream, typename OutputStream>
    static RAPIDJSON_FORCEINLINE void ScanCopyUnescapedString(InputStream&, OutputStream&) {
            // 对于通用版本，不执行任何操作
    }

// 如果定义了 SSE2 或 SSE42，则执行以下代码
#if defined(RAPIDJSON_SSE2) || defined(RAPIDJSON_SSE42)
    // 将 StringStream 转换为 StackStream<char>
        // 使用 SIMD 指令扫描和复制未转义的字符串
        static RAPIDJSON_FORCEINLINE void ScanCopyUnescapedString(StringStream& is, StackStream<char>& os) {
            const char* p = is.src_;

            // 逐个扫描字符，直到对齐（未对齐的加载可能跨页导致崩溃）
            const char* nextAligned = reinterpret_cast<const char*>((reinterpret_cast<size_t>(p) + 15) & static_cast<size_t>(~15));
            while (p != nextAligned)
                if (RAPIDJSON_UNLIKELY(*p == '\"') || RAPIDJSON_UNLIKELY(*p == '\\') || RAPIDJSON_UNLIKELY(static_cast<unsigned>(*p) < 0x20)) {
                    is.src_ = p;
                    return;
                }
                else
                    os.Put(*p++);

            // 使用 SIMD 处理剩余的字符串
            static const char dquote[16] = { '\"', '\"', '\"', '\"', '\"', '\"', '\"', '\"', '\"', '\"', '\"', '\"', '\"', '\"', '\"', '\"' };
            static const char bslash[16] = { '\\', '\\', '\\', '\\', '\\', '\\', '\\', '\\', '\\', '\\', '\\', '\\', '\\', '\\', '\\', '\\' };
            static const char space[16]  = { 0x1F, 0x1F, 0x1F, 0x1F, 0x1F, 0x1F, 0x1F, 0x1F, 0x1F, 0x1F, 0x1F, 0x1F, 0x1F, 0x1F, 0x1F, 0x1F };
            const __m128i dq = _mm_loadu_si128(reinterpret_cast<const __m128i *>(&dquote[0]));
            const __m128i bs = _mm_loadu_si128(reinterpret_cast<const __m128i *>(&bslash[0]));
            const __m128i sp = _mm_loadu_si128(reinterpret_cast<const __m128i *>(&space[0]));

            for (;; p += 16) {
                const __m128i s = _mm_load_si128(reinterpret_cast<const __m128i *>(p));
                const __m128i t1 = _mm_cmpeq_epi8(s, dq);
                const __m128i t2 = _mm_cmpeq_epi8(s, bs);
                const __m128i t3 = _mm_cmpeq_epi8(_mm_max_epu8(s, sp), sp); // s < 0x20 <=> max(s, 0x1F) == 0x1F
                const __m128i x = _mm_or_si128(_mm_or_si128(t1, t2), t3);
                unsigned short r = static_cast<unsigned short>(_mm_movemask_epi8(x));
                if (RAPIDJSON_UNLIKELY(r != 0)) {   // 一些字符被转义
                    SizeType length;
    #ifdef _MSC_VER         // 如果是在 MSVC 编译器下
                    unsigned long offset;  // 定义一个无符号长整型变量 offset
                    _BitScanForward(&offset, r);  // 使用 _BitScanForward 函数找到 r 中第一个为 1 的位的索引，存储到 offset 中
                    length = offset;  // 将 offset 赋值给 length
    #else
                    length = static_cast<SizeType>(__builtin_ffs(r) - 1);  // 如果不是在 MSVC 编译器下，使用 __builtin_ffs 函数找到 r 中第一个为 1 的位的索引，减 1 后赋值给 length
    #endif
                    if (length != 0) {  // 如果 length 不为 0
                        char* q = reinterpret_cast<char*>(os.Push(length));  // 将 os 中的长度为 length 的空间转换为字符指针 q
                        for (size_t i = 0; i < length; i++)  // 遍历 length 次
                            q[i] = p[i];  // 将 p 中的数据复制到 q 中
    
                        p += length;  // 将 p 向后移动 length 个位置
                    }
                    break;  // 跳出循环
                }
                _mm_storeu_si128(reinterpret_cast<__m128i *>(os.Push(16)), s);  // 将 s 存储到 os 中长度为 16 的空间
            }
    
            is.src_ = p;  // 将 p 赋值给 is.src_
        }
    
        // InsituStringStream -> InsituStringStream  // 注释：将 InsituStringStream 转换为 InsituStringStream
#ifdef _MSC_VER         // 如果是在 MSC 编译器下
                unsigned long offset;  // 定义一个无符号长整型变量 offset
                _BitScanForward(&offset, r);  // 使用 _BitScanForward 函数找到 r 中第一个置位的位置，存储在 offset 中
                length = offset;  // 将 offset 的值赋给 length
#else
                length = static_cast<size_t>(__builtin_ffs(r) - 1);  // 如果不是在 MSC 编译器下，使用 __builtin_ffs 函数找到 r 中第一个置位的位置，然后减去 1，将结果转换为 size_t 类型，赋给 length
#endif
                for (const char* pend = p + length; p != pend; )  // 循环，直到 p 和 pend 相等
                    *q++ = *p++;  // 将 p 指向的字符赋给 q 指向的位置，然后递增 p 和 q
                break;  // 跳出循环
            }
            _mm_storeu_si128(reinterpret_cast<__m128i *>(q), s);  // 使用 _mm_storeu_si128 函数将 s 存储到 q 指向的位置
        }

        is.src_ = p;  // 将 p 的值赋给 is 的 src_ 成员变量
        is.dst_ = q;  // 将 q 的值赋给 is 的 dst_ 成员变量
    }

    // 当读写指针在原地流中相同时，跳过未转义的字符
        // 跳过未转义的字符串
        static RAPIDJSON_FORCEINLINE void SkipUnescapedString(InsituStringStream& is) {
            // 断言源和目标指针相等
            RAPIDJSON_ASSERT(is.src_ == is.dst_);
            char* p = is.src_;

            // 逐个扫描字符，直到对齐（未对齐的加载可能会跨页导致崩溃）
            const char* nextAligned = reinterpret_cast<const char*>((reinterpret_cast<size_t>(p) + 15) & static_cast<size_t>(~15));
            for (; p != nextAligned; p++)
                // 如果遇到双引号、反斜杠或者控制字符（小于0x20），则更新源和目标指针并返回
                if (RAPIDJSON_UNLIKELY(*p == '\"') || RAPIDJSON_UNLIKELY(*p == '\\') || RAPIDJSON_UNLIKELY(static_cast<unsigned>(*p) < 0x20)) {
                    is.src_ = is.dst_ = p;
                    return;
                }

            // 使用 SIMD 处理剩余的字符串
            static const char dquote[16] = { '\"', '\"', '\"', '\"', '\"', '\"', '\"', '\"', '\"', '\"', '\"', '\"', '\"', '\"', '\"', '\"' };
            static const char bslash[16] = { '\\', '\\', '\\', '\\', '\\', '\\', '\\', '\\', '\\', '\\', '\\', '\\', '\\', '\\', '\\', '\\' };
            static const char space[16] = { 0x1F, 0x1F, 0x1F, 0x1F, 0x1F, 0x1F, 0x1F, 0x1F, 0x1F, 0x1F, 0x1F, 0x1F, 0x1F, 0x1F, 0x1F, 0x1F };
            const __m128i dq = _mm_loadu_si128(reinterpret_cast<const __m128i *>(&dquote[0]));
            const __m128i bs = _mm_loadu_si128(reinterpret_cast<const __m128i *>(&bslash[0]));
            const __m128i sp = _mm_loadu_si128(reinterpret_cast<const __m128i *>(&space[0]));

            for (;; p += 16) {
                const __m128i s = _mm_load_si128(reinterpret_cast<const __m128i *>(p));
                const __m128i t1 = _mm_cmpeq_epi8(s, dq);
                const __m128i t2 = _mm_cmpeq_epi8(s, bs);
                const __m128i t3 = _mm_cmpeq_epi8(_mm_max_epu8(s, sp), sp); // s < 0x20 <=> max(s, 0x1F) == 0x1F
                const __m128i x = _mm_or_si128(_mm_or_si128(t1, t2), t3);
                unsigned short r = static_cast<unsigned short>(_mm_movemask_epi8(x));
                // 如果存在转义字符，则跳出循环
                if (RAPIDJSON_UNLIKELY(r != 0)) {   // some of characters is escaped
                    size_t length;
#ifdef _MSC_VER         // 如果是在 MSC 编译器下
                unsigned long offset;  // 定义一个无符号长整型变量 offset
                _BitScanForward(&offset, r);  // 使用 _BitScanForward 函数找到 r 中第一个被转义的字符的索引，存储在 offset 中
                length = offset;  // 将 offset 赋值给 length
#else
                length = static_cast<size_t>(__builtin_ffs(r) - 1);  // 如果不是在 MSC 编译器下，使用 __builtin_ffs 函数找到 r 中第一个被转义的字符的索引，减去 1 后赋值给 length
#endif
                p += length;  // 将指针 p 向后移动 length 个位置
                break;  // 跳出循环
            }
        }

        is.src_ = is.dst_ = p;  // 将 is 的源指针和目标指针都指向 p
    }
#elif defined(RAPIDJSON_NEON)
    // StringStream -> StackStream<char>  // 将 StringStream 转换为 StackStream<char>
    }

    // InsituStringStream -> InsituStringStream  // 将 InsituStringStream 转换为 InsituStringStream
    }

    // 当读/写指针在原地流中相同时，跳过未转义的字符
    // 跳过未转义的字符串，修改输入输出流的指针位置
    static RAPIDJSON_FORCEINLINE void SkipUnescapedString(InsituStringStream& is) {
        // 断言输入输出流的指针位置相同
        RAPIDJSON_ASSERT(is.src_ == is.dst_);
        char* p = is.src_;

        // 逐个扫描字符，直到对齐（未对齐的加载可能跨越页面边界并导致崩溃）
        const char* nextAligned = reinterpret_cast<const char*>((reinterpret_cast<size_t>(p) + 15) & static_cast<size_t>(~15));
        for (; p != nextAligned; p++)
            // 如果遇到双引号、反斜杠或者控制字符，则更新输入输出流的指针位置并返回
            if (RAPIDJSON_UNLIKELY(*p == '\"') || RAPIDJSON_UNLIKELY(*p == '\\') || RAPIDJSON_UNLIKELY(static_cast<unsigned>(*p) < 0x20)) {
                is.src_ = is.dst_ = p;
                return;
            }

        // 使用 SIMD 处理剩余的字符串
        const uint8x16_t s0 = vmovq_n_u8('"');
        const uint8x16_t s1 = vmovq_n_u8('\\');
        const uint8x16_t s2 = vmovq_n_u8('\b');
        const uint8x16_t s3 = vmovq_n_u8(32);

        for (;; p += 16) {
            const uint8x16_t s = vld1q_u8(reinterpret_cast<uint8_t *>(p));
            uint8x16_t x = vceqq_u8(s, s0);
            x = vorrq_u8(x, vceqq_u8(s, s1));
            x = vorrq_u8(x, vceqq_u8(s, s2));
            x = vorrq_u8(x, vcltq_u8(s, s3));

            x = vrev64q_u8(x);                     // 反转 64 位
            uint64_t low = vgetq_lane_u64(vreinterpretq_u64_u8(x), 0);   // 提取低位
            uint64_t high = vgetq_lane_u64(vreinterpretq_u64_u8(x), 1);  // 提取高位

            if (low == 0) {
                if (high != 0) {
                    uint32_t lz = internal::clzll(high);
                    p += 8 + (lz >> 3);
                    break;
                }
            } else {
                uint32_t lz = internal::clzll(low);
                p += lz >> 3;
                break;
            }
        }

        // 更新输入输出流的指针位置
        is.src_ = is.dst_ = p;
    }
#endif // RAPIDJSON_NEON

// 定义一个模板类 NumberStream，用于处理输入流中的数字字符
template<typename InputStream, typename StackCharacter, bool backup, bool pushOnTake>
class NumberStream;

// 针对不需要备份和推送的情况进行特化
template<typename InputStream, typename StackCharacter>
class NumberStream<InputStream, StackCharacter, false, false> {
public:
    typedef typename InputStream::Ch Ch;

    // 构造函数，初始化输入流
    NumberStream(GenericReader& reader, InputStream& s) : is(s) { (void)reader;  }

    // 返回下一个字符但不移动读取位置
    RAPIDJSON_FORCEINLINE Ch Peek() const { return is.Peek(); }
    // 返回下一个字符并移动读取位置
    RAPIDJSON_FORCEINLINE Ch TakePush() { return is.Take(); }
    // 返回下一个字符并移动读取位置
    RAPIDJSON_FORCEINLINE Ch Take() { return is.Take(); }
    // 推送一个字符到栈中
    RAPIDJSON_FORCEINLINE void Push(char) {}

    // 返回当前读取位置
    size_t Tell() { return is.Tell(); }
    // 返回栈的长度
    size_t Length() { return 0; }
    // 弹出栈顶元素
    const StackCharacter* Pop() { return 0; }

protected:
    // 禁止赋值操作
    NumberStream& operator=(const NumberStream&);

    // 输入流对象
    InputStream& is;
};

// 针对需要备份但不需要推送的情况进行特化
template<typename InputStream, typename StackCharacter>
class NumberStream<InputStream, StackCharacter, true, false> : public NumberStream<InputStream, StackCharacter, false, false> {
    typedef NumberStream<InputStream, StackCharacter, false, false> Base;
public:
    // 构造函数，初始化输入流和栈流
    NumberStream(GenericReader& reader, InputStream& is) : Base(reader, is), stackStream(reader.stack_) {}

    // 返回下一个字符并将其推送到栈中
    RAPIDJSON_FORCEINLINE Ch TakePush() {
        stackStream.Put(static_cast<StackCharacter>(Base::is.Peek()));
        return Base::is.Take();
    }

    // 推送一个字符到栈中
    RAPIDJSON_FORCEINLINE void Push(StackCharacter c) {
        stackStream.Put(c);
    }

    // 返回栈的长度
    size_t Length() { return stackStream.Length(); }

    // 弹出栈顶元素
    const StackCharacter* Pop() {
        stackStream.Put('\0');
        return stackStream.Pop();
    }

private:
    // 栈流对象
    StackStream<StackCharacter> stackStream;
};

// 继续定义模板类 NumberStream
template<typename InputStream, typename StackCharacter>
    // 定义一个模板类 NumberStream，继承自具有不同模板参数的 NumberStream 类
    // 模板参数分别为 InputStream, StackCharacter, true, true
    template<unsigned parseFlags, typename InputStream, typename Handler>
    // 定义一个模板类 NumberStream，继承自具有不同模板参数的 NumberStream 类
    // 模板参数分别为 InputStream, StackCharacter, true, true
    class NumberStream<InputStream, StackCharacter, true, true> : public NumberStream<InputStream, StackCharacter, true, false> {
        // 使用 typedef 定义别名 Base，指向具有模板参数 InputStream, StackCharacter, true, false 的 NumberStream 类
        typedef NumberStream<InputStream, StackCharacter, true, false> Base;
    public:
        // 构造函数，接受 GenericReader 类型的引用和 InputStream 类型的引用
        NumberStream(GenericReader& reader, InputStream& is) : Base(reader, is) {}

        // 定义一个内联函数 Take，返回类型为 Ch，调用 Base 类的 TakePush 函数
        RAPIDJSON_FORCEINLINE Ch Take() { return Base::TakePush(); }
    };
#if RAPIDJSON_64BIT
                // 如果是 64 位架构，使用 i64 存储 64 位架构中的有效数字
                if (!use64bit)
                    i64 = i;

                while (RAPIDJSON_LIKELY(s.Peek() >= '0' && s.Peek() <= '9')) {
                    // 如果 i64 大于 2^53 - 1，则退出循环
                    if (i64 > RAPIDJSON_UINT64_C2(0x1FFFFF, 0xFFFFFFFF)) // 2^53 - 1 for fast path
                        break;
                    else {
                        // 将字符转换为数字，计算有效数字
                        i64 = i64 * 10 + static_cast<unsigned>(s.TakePush() - '0');
                        --expFrac;
                        if (i64 != 0)
                            significandDigit++;
                    }
                }

                // 将 i64 转换为 double 类型
                d = static_cast<double>(i64);
#else
                // 如果不是 64 位架构，使用 double 存储有效数字
                d = static_cast<double>(use64bit ? i64 : i);
    }

    // 解析任何 JSON 值
    template<unsigned parseFlags, typename InputStream, typename Handler>
    void ParseValue(InputStream& is, Handler& handler) {
        switch (is.Peek()) {
            case 'n': ParseNull  <parseFlags>(is, handler); break;
            case 't': ParseTrue  <parseFlags>(is, handler); break;
            case 'f': ParseFalse <parseFlags>(is, handler); break;
            case '"': ParseString<parseFlags>(is, handler); break;
            case '{': ParseObject<parseFlags>(is, handler); break;
            case '[': ParseArray <parseFlags>(is, handler); break;
            default :
                      // 解析数字
                      ParseNumber<parseFlags>(is, handler);
                      break;

        }
    }

    // 迭代解析

    // 状态
    // 定义枚举类型 IterativeParsingState，表示迭代解析的状态
    enum IterativeParsingState {
        IterativeParsingFinishState = 0, // sink states at top
        IterativeParsingErrorState,      // sink states at top
        IterativeParsingStartState,

        // Object states
        IterativeParsingObjectInitialState,
        IterativeParsingMemberKeyState,
        IterativeParsingMemberValueState,
        IterativeParsingObjectFinishState,

        // Array states
        IterativeParsingArrayInitialState,
        IterativeParsingElementState,
        IterativeParsingArrayFinishState,

        // Single value state
        IterativeParsingValueState,

        // Delimiter states (at bottom)
        IterativeParsingElementDelimiterState,
        IterativeParsingMemberDelimiterState,
        IterativeParsingKeyValueDelimiterState,

        cIterativeParsingStateCount
    };

    // 定义枚举类型 Token，表示不同的标记类型
    enum Token {
        LeftBracketToken = 0,
        RightBracketToken,

        LeftCurlyBracketToken,
        RightCurlyBracketToken,

        CommaToken,
        ColonToken,

        StringToken,
        FalseToken,
        TrueToken,
        NullToken,
        NumberToken,

        kTokenCount
    };

    // 定义 Tokenize 函数，用于将字符转换为标记类型
    RAPIDJSON_FORCEINLINE Token Tokenize(Ch c) const {
//!@cond RAPIDJSON_HIDDEN_FROM_DOXYGEN
// 定义 NumberToken 为 N
#define N NumberToken
// 定义 N16 为 16 个 N
#define N16 N,N,N,N,N,N,N,N,N,N,N,N,N,N,N,N
        // 从 ASCII 到 Token 的映射表
        static const unsigned char tokenMap[256] = {
            N16, // 00~0F
            N16, // 10~1F
            N, N, StringToken, N, N, N, N, N, N, N, N, N, CommaToken, N, N, N, // 20~2F
            N, N, N, N, N, N, N, N, N, N, ColonToken, N, N, N, N, N, // 30~3F
            N16, // 40~4F
            N, N, N, N, N, N, N, N, N, N, N, LeftBracketToken, N, RightBracketToken, N, N, // 50~5F
            N, N, N, N, N, N, FalseToken, N, N, N, N, N, N, N, NullToken, N, // 60~6F
            N, N, N, N, TrueToken, N, N, N, N, N, N, LeftCurlyBracketToken, N, RightCurlyBracketToken, N, N, // 70~7F
            N16, N16, N16, N16, N16, N16, N16, N16 // 80~FF
        };
// 取消定义 N
#undef N
// 取消定义 N16
#undef N16
//!@endcond

// 如果 Ch 的大小为 1 或者 c 的值小于 256
if (sizeof(Ch) == 1 || static_cast<unsigned>(c) < 256)
    // 返回 tokenMap 中对应位置的 Token
    return static_cast<Token>(tokenMap[static_cast<unsigned char>(c)]);
else
    // 返回 NumberToken
    return NumberToken;
}

}

// 根据候选目标状态在标记流和状态上进行推进。
// 可能返回状态弹出时的新状态。
template <unsigned parseFlags, typename InputStream, typename Handler>
}

template <typename InputStream>
    // 处理解析错误的函数
    void HandleError(IterativeParsingState src, InputStream& is) {
        // 如果已经设置了解析错误标志，则直接返回
        if (HasParseError()) {
            return;
        }

        // 根据不同的解析状态，抛出相应的解析错误
        switch (src) {
        case IterativeParsingStartState:            RAPIDJSON_PARSE_ERROR(kParseErrorDocumentEmpty, is.Tell()); return;
        case IterativeParsingFinishState:           RAPIDJSON_PARSE_ERROR(kParseErrorDocumentRootNotSingular, is.Tell()); return;
        case IterativeParsingObjectInitialState:
        case IterativeParsingMemberDelimiterState:  RAPIDJSON_PARSE_ERROR(kParseErrorObjectMissName, is.Tell()); return;
        case IterativeParsingMemberKeyState:        RAPIDJSON_PARSE_ERROR(kParseErrorObjectMissColon, is.Tell()); return;
        case IterativeParsingMemberValueState:      RAPIDJSON_PARSE_ERROR(kParseErrorObjectMissCommaOrCurlyBracket, is.Tell()); return;
        case IterativeParsingKeyValueDelimiterState:
        case IterativeParsingArrayInitialState:
        case IterativeParsingElementDelimiterState: RAPIDJSON_PARSE_ERROR(kParseErrorValueInvalid, is.Tell()); return;
        default: RAPIDJSON_ASSERT(src == IterativeParsingElementState); RAPIDJSON_PARSE_ERROR(kParseErrorArrayMissCommaOrSquareBracket, is.Tell()); return;
        }
    }

    // 判断是否为迭代解析的分隔符状态
    RAPIDJSON_FORCEINLINE bool IsIterativeParsingDelimiterState(IterativeParsingState s) const {
        return s >= IterativeParsingElementDelimiterState;
    }

    // 判断是否为迭代解析的完成状态
    RAPIDJSON_FORCEINLINE bool IsIterativeParsingCompleteState(IterativeParsingState s) const {
        return s <= IterativeParsingErrorState;
    }

    template <unsigned parseFlags, typename InputStream, typename Handler>
    // 通过迭代方式解析输入流，返回解析结果
    ParseResult IterativeParse(InputStream& is, Handler& handler) {
        // 清空解析结果
        parseResult_.Clear();
        // 在退出时清空堆栈
        ClearStackOnExit scope(*this);
        // 初始化迭代解析状态
        IterativeParsingState state = IterativeParsingStartState;

        // 跳过空白字符和注释
        SkipWhitespaceAndComments<parseFlags>(is);
        // 检查并返回解析错误
        RAPIDJSON_PARSE_ERROR_EARLY_RETURN(parseResult_);
        // 当输入流未结束时循环解析
        while (is.Peek() != '\0') {
            // 获取当前字符的标记
            Token t = Tokenize(is.Peek());
            // 预测下一个状态
            IterativeParsingState n = Predict(state, t);
            // 进行状态转移并返回新状态
            IterativeParsingState d = Transit<parseFlags>(state, t, n, is, handler);

            // 如果状态为错误状态，处理错误并跳出循环
            if (d == IterativeParsingErrorState) {
                HandleError(state, is);
                break;
            }

            // 更新状态
            state = d;

            // 如果解析标志包含停止解析标志，并且状态为完成状态，则跳出循环
            if ((parseFlags & kParseStopWhenDoneFlag) && state == IterativeParsingFinishState)
                break;

            // 跳过空白字符和注释
            SkipWhitespaceAndComments<parseFlags>(is);
            // 检查并返回解析错误
            RAPIDJSON_PARSE_ERROR_EARLY_RETURN(parseResult_);
        }

        // 处理文件结束
        if (state != IterativeParsingFinishState)
            HandleError(state, is);

        // 返回解析结果
        return parseResult_;
    }

    // 默认堆栈容量
    static const size_t kDefaultStackCapacity = 256;    //!< Default stack capacity in bytes for storing a single decoded string.
    // 用于临时存储解码字符串的堆栈
    internal::Stack<StackAllocator> stack_;  //!< A stack for storing decoded string temporarily during non-destructive parsing.
    // 解析结果
    ParseResult parseResult_;
    // 迭代解析状态
    IterativeParsingState state_;
}; // class GenericReader

//! Reader with UTF8 encoding and default allocator.
typedef GenericReader<UTF8<>, UTF8<> > Reader;

RAPIDJSON_NAMESPACE_END

#if defined(__clang__) || defined(_MSC_VER)
RAPIDJSON_DIAG_POP
#endif


#ifdef __GNUC__
RAPIDJSON_DIAG_POP
#endif

#endif // RAPIDJSON_READER_H_
```