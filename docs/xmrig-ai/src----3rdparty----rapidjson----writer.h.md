# `xmrig\src\3rdparty\rapidjson\writer.h`

```
// 定义了 RapidJSON 的写入器 Writer 类
#ifndef RAPIDJSON_WRITER_H_
#define RAPIDJSON_WRITER_H_

// 包含了流处理相关的头文件
#include "stream.h"
// 包含了计算整数的头文件
#include "internal/clzll.h"
// 包含了元数据相关的头文件
#include "internal/meta.h"
// 包含了堆栈相关的头文件
#include "internal/stack.h"
// 包含了字符串处理相关的头文件
#include "internal/strfunc.h"
// 包含了双精度浮点数转字符串相关的头文件
#include "internal/dtoa.h"
// 包含了整数转字符串相关的头文件
#include "internal/itoa.h"
// 包含了字符串缓冲区相关的头文件
#include "stringbuffer.h"
// 包含了新的头文件
#include <new>      // placement new

// 如果定义了 RAPIDJSON_SIMD 并且是 MSC 编译器，则包含内联汇编头文件
#if defined(RAPIDJSON_SIMD) && defined(_MSC_VER)
#include <intrin.h>
#pragma intrinsic(_BitScanForward)
#endif
// 如果定义了 RAPIDJSON_SSE42，则包含 SSE4.2 相关头文件
#ifdef RAPIDJSON_SSE42
#include <nmmintrin.h>
// 如果定义了 RAPIDJSON_SSE2，则包含 SSE2 相关头文件
#elif defined(RAPIDJSON_SSE2)
#include <emmintrin.h>
// 如果定义了 RAPIDJSON_NEON，则包含 NEON 相关头文件
#elif defined(RAPIDJSON_NEON)
#include <arm_neon.h>
#endif

// 如果是 clang 编译器，则关闭部分警告
#ifdef __clang__
RAPIDJSON_DIAG_PUSH
RAPIDJSON_DIAG_OFF(padded)
RAPIDJSON_DIAG_OFF(unreachable-code)
RAPIDJSON_DIAG_OFF(c++98-compat)
// 如果是 MSC 编译器，则关闭部分警告
#elif defined(_MSC_VER)
RAPIDJSON_DIAG_PUSH
RAPIDJSON_DIAG_OFF(4127) // conditional expression is constant
#endif

// RapidJSON 命名空间开始
RAPIDJSON_NAMESPACE_BEGIN

///////////////////////////////////////////////////////////////////////////////
// WriteFlag

/*! \def RAPIDJSON_WRITE_DEFAULT_FLAGS
    \ingroup RAPIDJSON_CONFIG
    \brief User-defined kWriteDefaultFlags definition.

    User can define this as any \c WriteFlag combinations.
*/
// 如果未定义 RAPIDJSON_WRITE_DEFAULT_FLAGS，则定义为 kWriteNoFlags
#ifndef RAPIDJSON_WRITE_DEFAULT_FLAGS
#define RAPIDJSON_WRITE_DEFAULT_FLAGS kWriteNoFlags
#endif

//! Combination of writeFlags
// 写入标志的组合
enum WriteFlag {
    kWriteNoFlags = 0,              //!< No flags are set.  // 设置为0，表示没有任何标志位被设置
    kWriteValidateEncodingFlag = 1, //!< Validate encoding of JSON strings.  // 设置为1，表示验证JSON字符串的编码
    kWriteNanAndInfFlag = 2,        //!< Allow writing of Infinity, -Infinity and NaN.  // 设置为2，表示允许写入Infinity、-Infinity和NaN
    kWriteDefaultFlags = RAPIDJSON_WRITE_DEFAULT_FLAGS  //!< Default write flags. Can be customized by defining RAPIDJSON_WRITE_DEFAULT_FLAGS  // 设置为RAPIDJSON_WRITE_DEFAULT_FLAGS，表示默认的写入标志位，可以通过定义RAPIDJSON_WRITE_DEFAULT_FLAGS来自定义
};

//! JSON writer
/*! Writer implements the concept Handler.
    It generates JSON text by events to an output os.

    User may programmatically calls the functions of a writer to generate JSON text.

    On the other side, a writer can also be passed to objects that generates events,

    for example Reader::Parse() and Document::Accept().

    \tparam OutputStream Type of output stream.
    \tparam SourceEncoding Encoding of source string.
    \tparam TargetEncoding Encoding of output stream.
    \tparam StackAllocator Type of allocator for allocating memory of stack.
    \note implements Handler concept
*/
template<typename OutputStream, typename SourceEncoding = UTF8<>, typename TargetEncoding = UTF8<>, typename StackAllocator = CrtAllocator, unsigned writeFlags = kWriteDefaultFlags>
class Writer {
public:
    typedef typename SourceEncoding::Ch Ch;

    static const int kDefaultMaxDecimalPlaces = 324;

    //! Constructor
    /*! \param os Output stream.
        \param stackAllocator User supplied allocator. If it is null, it will create a private one.
        \param levelDepth Initial capacity of stack.
    */
    explicit
    Writer(OutputStream& os, StackAllocator* stackAllocator = 0, size_t levelDepth = kDefaultLevelDepth) :
        os_(&os), level_stack_(stackAllocator, levelDepth * sizeof(Level)), maxDecimalPlaces_(kDefaultMaxDecimalPlaces), hasRoot_(false) {}

    explicit
    Writer(StackAllocator* allocator = 0, size_t levelDepth = kDefaultLevelDepth) :
        os_(0), level_stack_(allocator, levelDepth * sizeof(Level)), maxDecimalPlaces_(kDefaultMaxDecimalPlaces), hasRoot_(false) {}

#if RAPIDJSON_HAS_CXX11_RVALUE_REFS
    Writer(Writer&& rhs) :
        os_(rhs.os_), level_stack_(std::move(rhs.level_stack_)), maxDecimalPlaces_(rhs.maxDecimalPlaces_), hasRoot_(rhs.hasRoot_) {
        rhs.os_ = 0;
    }
#endif

    //! Reset the writer with a new stream.
    /*!
        This function reset the writer with a new stream and default settings,
        in order to make a Writer object reusable for output multiple JSONs.

        \param os New output stream.
        \code
        Writer<OutputStream> writer(os1);
        writer.StartObject();
        // ...
        writer.EndObject();

        writer.Reset(os2);
        writer.StartObject();
        // ...
        writer.EndObject();
        \endcode
    */
    void Reset(OutputStream& os) {
        os_ = &os;  // 设置新的输出流
        hasRoot_ = false;  // 重置根节点标志
        level_stack_.Clear();  // 清空层级栈
    }

    //! Checks whether the output is a complete JSON.
    /*!
        A complete JSON has a complete root object or array.
    */
    bool IsComplete() const {
        return hasRoot_ && level_stack_.Empty();  // 判断输出是否为完整的 JSON
    }

    int GetMaxDecimalPlaces() const {
        return maxDecimalPlaces_;  // 获取最大小数位数设置
    }

    //! Sets the maximum number of decimal places for double output.
    /*!
        This setting truncates the output with specified number of decimal places.

        For example,

        \code
        writer.SetMaxDecimalPlaces(3);
        writer.StartArray();
        writer.Double(0.12345);                 // "0.123"
        writer.Double(0.0001);                  // "0.0"
        writer.Double(1.234567890123456e30);    // "1.234567890123456e30" (do not truncate significand for positive exponent)
        writer.Double(1.23e-4);                 // "0.0"                  (do truncate significand for negative exponent)
        writer.EndArray();
        \endcode

        The default setting does not truncate any decimal places. You can restore to this setting by calling
        \code
        writer.SetMaxDecimalPlaces(Writer::kDefaultMaxDecimalPlaces);
        \endcode
    */
    void SetMaxDecimalPlaces(int maxDecimalPlaces) {
        maxDecimalPlaces_ = maxDecimalPlaces;  // 设置最大小数位数
    }

    /*!@name Implementation of Handler
        \see Handler
    */
    //@{
    # 返回空值类型
    bool Null()                 { Prefix(kNullType);   return EndValue(WriteNull()); }
    # 返回布尔类型，根据参数值确定是 true 还是 false
    bool Bool(bool b)           { Prefix(b ? kTrueType : kFalseType); return EndValue(WriteBool(b)); }
    # 返回整数类型，参数为 int 类型
    bool Int(int i)             { Prefix(kNumberType); return EndValue(WriteInt(i)); }
    # 返回无符号整数类型，参数为 unsigned 类型
    bool Uint(unsigned u)       { Prefix(kNumberType); return EndValue(WriteUint(u)); }
    # 返回 64 位整数类型，参数为 int64_t 类型
    bool Int64(int64_t i64)     { Prefix(kNumberType); return EndValue(WriteInt64(i64)); }
    # 返回 64 位无符号整数类型，参数为 uint64_t 类型
    bool Uint64(uint64_t u64)   { Prefix(kNumberType); return EndValue(WriteUint64(u64)); }

    # 写入给定的双精度浮点数值到流中
    # 参数 d：要写入的值
    # 返回是否成功
    bool Double(double d)       { Prefix(kNumberType); return EndValue(WriteDouble(d)); }

    # 写入原始数字类型，参数为字符串指针、长度和是否复制标志
    bool RawNumber(const Ch* str, SizeType length, bool copy = false) {
        # 断言字符串指针不为空
        RAPIDJSON_ASSERT(str != 0);
        # 忽略复制标志
        (void)copy;
        # 设置前缀为数字类型
        Prefix(kNumberType);
        # 返回写入字符串结果
        return EndValue(WriteString(str, length));
    }

    # 写入字符串类型，参数为字符串指针、长度和是否复制标志
    bool String(const Ch* str, SizeType length, bool copy = false) {
        # 断言字符串指针不为空
        RAPIDJSON_ASSERT(str != 0);
        # 忽略复制标志
        (void)copy;
        # 设置前缀为字符串类型
        Prefix(kStringType);
        # 返回写入字符串结果
        return EndValue(WriteString(str, length));
    }
#if RAPIDJSON_HAS_STDSTRING
    # 如果定义了RAPIDJSON_HAS_STDSTRING，则定义String方法，接受std::basic_string<Ch>类型的参数
    bool String(const std::basic_string<Ch>& str) {
        # 调用String方法，传入str的数据和长度
        return String(str.data(), SizeType(str.size()));
    }
#endif

    # 开始一个JSON对象
    bool StartObject() {
        # 在JSON中添加一个对象前缀
        Prefix(kObjectType);
        # 在level_stack_中创建一个新的Level对象，表示当前不在数组中
        new (level_stack_.template Push<Level>()) Level(false);
        # 写入JSON对象的开始
        return WriteStartObject();
    }

    # 添加一个JSON对象的键
    bool Key(const Ch* str, SizeType length, bool copy = false) { return String(str, length, copy); }

#if RAPIDJSON_HAS_STDSTRING
    # 如果定义了RAPIDJSON_HAS_STDSTRING，则定义Key方法，接受std::basic_string<Ch>类型的参数
    bool Key(const std::basic_string<Ch>& str)
    {
      # 调用Key方法，传入str的数据和长度
      return Key(str.data(), SizeType(str.size()));
    }
#endif

    # 结束一个JSON对象
    bool EndObject(SizeType memberCount = 0) {
        (void)memberCount;
        # 断言：level_stack_的大小大于等于Level对象的大小，表示不在对象内部
        RAPIDJSON_ASSERT(level_stack_.GetSize() >= sizeof(Level)); // not inside an Object
        # 断言：当前不在数组中，而是在对象中
        RAPIDJSON_ASSERT(!level_stack_.template Top<Level>()->inArray); // currently inside an Array, not Object
        # 断言：对象的键和值的数量是偶数
        RAPIDJSON_ASSERT(0 == level_stack_.template Top<Level>()->valueCount % 2); // Object has a Key without a Value
        # 从level_stack_中弹出一个Level对象
        level_stack_.template Pop<Level>(1);
        # 结束值的写入，并返回结束对象的结果
        return EndValue(WriteEndObject());
    }

    # 开始一个JSON数组
    bool StartArray() {
        # 在JSON中添加一个数组前缀
        Prefix(kArrayType);
        # 在level_stack_中创建一个新的Level对象，表示当前在数组中
        new (level_stack_.template Push<Level>()) Level(true);
        # 写入JSON数组的开始
        return WriteStartArray();
    }

    # 结束一个JSON数组
    bool EndArray(SizeType elementCount = 0) {
        (void)elementCount;
        # 断言：level_stack_的大小大于等于Level对象的大小，表示不在数组内部
        RAPIDJSON_ASSERT(level_stack_.GetSize() >= sizeof(Level));
        # 断言：当前在数组中
        RAPIDJSON_ASSERT(level_stack_.template Top<Level>()->inArray);
        # 从level_stack_中弹出一个Level对象
        level_stack_.template Pop<Level>(1);
        # 结束值的写入，并返回结束数组的结果
        return EndValue(WriteEndArray());
    }
    //@}

    /*! @name Convenience extensions */
    //@{

    //! Simpler but slower overload.
    # 以更简单但更慢的方式重载String方法，接受Ch*类型的参数
    bool String(const Ch* const& str) { return String(str, internal::StrLen(str)); }
    # 以更简单但更慢的方式重载Key方法，接受Ch*类型的参数
    bool Key(const Ch* const& str) { return Key(str, internal::StrLen(str)); }

    //@}

    //! Write a raw JSON value.
    # 为用户提供将字符串化的 JSON 作为值写入的函数
    # json: 一个格式良好的 JSON 值。它不应该在 [0, length - 1] 范围内包含空字符。
    # length: json 的长度
    # type: json 根节点的类型
    bool RawValue(const Ch* json, size_t length, Type type) {
        # 断言 json 不为空
        RAPIDJSON_ASSERT(json != 0);
        # 根据类型添加前缀
        Prefix(type);
        # 写入原始值并结束值
        return EndValue(WriteRawValue(json, length));
    }
    
    # 刷新输出流
    # 允许用户立即刷新输出流
    void Flush() {
        # 刷新输出流
        os_->Flush();
    }
    
    # 默认的层级深度
    static const size_t kDefaultLevelDepth = 32;
protected:
    //! Information for each nested level
    // 每个嵌套级别的信息
    struct Level {
        Level(bool inArray_) : valueCount(0), inArray(inArray_) {}
        size_t valueCount;  //!< number of values in this level
        // 当前级别中值的数量
        bool inArray;       //!< true if in array, otherwise in object
        // 如果在数组中则为 true，否则在对象中
        bool inLine = false;
    };

    bool WriteNull()  {
        // 写入空值
        PutReserve(*os_, 4);
        PutUnsafe(*os_, 'n'); PutUnsafe(*os_, 'u'); PutUnsafe(*os_, 'l'); PutUnsafe(*os_, 'l'); return true;
    }

    bool WriteBool(bool b)  {
        // 写入布尔值
        if (b) {
            PutReserve(*os_, 4);
            PutUnsafe(*os_, 't'); PutUnsafe(*os_, 'r'); PutUnsafe(*os_, 'u'); PutUnsafe(*os_, 'e');
        }
        else {
            PutReserve(*os_, 5);
            PutUnsafe(*os_, 'f'); PutUnsafe(*os_, 'a'); PutUnsafe(*os_, 'l'); PutUnsafe(*os_, 's'); PutUnsafe(*os_, 'e');
        }
        return true;
    }

    bool WriteInt(int i) {
        // 写入整数
        char buffer[11];
        const char* end = internal::i32toa(i, buffer);
        PutReserve(*os_, static_cast<size_t>(end - buffer));
        for (const char* p = buffer; p != end; ++p)
            PutUnsafe(*os_, static_cast<typename OutputStream::Ch>(*p));
        return true;
    }

    bool WriteUint(unsigned u) {
        // 写入无符号整数
        char buffer[10];
        const char* end = internal::u32toa(u, buffer);
        PutReserve(*os_, static_cast<size_t>(end - buffer));
        for (const char* p = buffer; p != end; ++p)
            PutUnsafe(*os_, static_cast<typename OutputStream::Ch>(*p));
        return true;
    }

    bool WriteInt64(int64_t i64) {
        // 写入64位整数
        char buffer[21];
        const char* end = internal::i64toa(i64, buffer);
        PutReserve(*os_, static_cast<size_t>(end - buffer));
        for (const char* p = buffer; p != end; ++p)
            PutUnsafe(*os_, static_cast<typename OutputStream::Ch>(*p));
        return true;
    }
    // 写入一个 64 位无符号整数到输出流中
    bool WriteUint64(uint64_t u64) {
        // 创建一个大小为 20 的字符数组作为缓冲区
        char buffer[20];
        // 将 64 位无符号整数转换为字符串，并将结果保存到缓冲区中
        char* end = internal::u64toa(u64, buffer);
        // 将转换后的字符串长度传递给 PutReserve 函数，用于预留输出流的空间
        PutReserve(*os_, static_cast<size_t>(end - buffer));
        // 将缓冲区中的内容逐个字符写入输出流
        for (char* p = buffer; p != end; ++p)
            PutUnsafe(*os_, static_cast<typename OutputStream::Ch>(*p));
        return true;
    }

    // 写入一个双精度浮点数到输出流中
    bool WriteDouble(double d) {
        // 检查浮点数是否为 NaN 或无穷大
        if (internal::Double(d).IsNanOrInf()) {
            // 如果不允许写入 NaN 和无穷大，则返回 false
            if (!(writeFlags & kWriteNanAndInfFlag))
                return false;
            // 如果浮点数为 NaN，则写入 "NaN" 到输出流
            if (internal::Double(d).IsNan()) {
                PutReserve(*os_, 3);
                PutUnsafe(*os_, 'N'); PutUnsafe(*os_, 'a'); PutUnsafe(*os_, 'N');
                return true;
            }
            // 如果浮点数为负无穷大，则写入 "-Infinity" 到输出流
            if (internal::Double(d).Sign()) {
                PutReserve(*os_, 9);
                PutUnsafe(*os_, '-');
            }
            else
                PutReserve(*os_, 8);
            PutUnsafe(*os_, 'I'); PutUnsafe(*os_, 'n'); PutUnsafe(*os_, 'f');
            PutUnsafe(*os_, 'i'); PutUnsafe(*os_, 'n'); PutUnsafe(*os_, 'i'); PutUnsafe(*os_, 't'); PutUnsafe(*os_, 'y');
            return true;
        }

        // 创建一个大小为 25 的字符数组作为缓冲区
        char buffer[25];
        // 将双精度浮点数转换为字符串，并将结果保存到缓冲区中
        char* end = internal::dtoa(d, buffer, maxDecimalPlaces_);
        // 将转换后的字符串长度传递给 PutReserve 函数，用于预留输出流的空间
        PutReserve(*os_, static_cast<size_t>(end - buffer));
        // 将缓冲区中的内容逐个字符写入输出流
        for (char* p = buffer; p != end; ++p)
            PutUnsafe(*os_, static_cast<typename OutputStream::Ch>(*p));
        return true;
    }

    // 写入一个字符串到输出流中
    bool WriteString(const Ch* str, SizeType length)  {
        // 创建一个包含十六进制数字的字符数组
        static const typename OutputStream::Ch hexDigits[16] = { '0', '1', '2', '3', '4', '5', '6', '7', '8', '9', 'A', 'B', 'C', 'D', 'E', 'F' };
        // 创建一个大小为 256 的字符数组，用于转义字符
        static const char escape[256] = {
#define Z16 0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0
            // 定义一个16个0的宏，用于后续的初始化
            //0    1    2    3    4    5    6    7    8    9    A    B    C    D    E    F
            'u', 'u', 'u', 'u', 'u', 'u', 'u', 'u', 'b', 't', 'n', 'u', 'f', 'r', 'u', 'u', // 00
            'u', 'u', 'u', 'u', 'u', 'u', 'u', 'u', 'u', 'u', 'u', 'u', 'u', 'u', 'u', 'u', // 10
              0,   0, '"',   0,   0,   0,   0,   0,   0,   0,   0,   0,   0,   0,   0,   0, // 20
            Z16, Z16,                                                                       // 30~4F
              0,   0,   0,   0,   0,   0,   0,   0,   0,   0,   0,   0,'\\',   0,   0,   0, // 50
            Z16, Z16, Z16, Z16, Z16, Z16, Z16, Z16, Z16, Z16                                // 60~FF
    }

    bool ScanWriteUnescapedString(GenericStringStream<SourceEncoding>& is, size_t length) {
        // 检查是否可以安全地写入未转义的字符串
        return RAPIDJSON_LIKELY(is.Tell() < length);
    }

    bool WriteStartObject() { os_->Put('{'); return true; }
    // 写入对象的起始符号 '{'
    bool WriteEndObject()   { os_->Put('}'); return true; }
    // 写入对象的结束符号 '}'
    bool WriteStartArray()  { os_->Put('['); return true; }
    // 写入数组的起始符号 '['
    bool WriteEndArray()    { os_->Put(']'); return true; }
    // 写入数组的结束符号 ']'

    bool WriteRawValue(const Ch* json, size_t length) {
        // 写入原始值，不做转义
        PutReserve(*os_, length);
        GenericStringStream<SourceEncoding> is(json);
        while (RAPIDJSON_LIKELY(is.Tell() < length)) {
            RAPIDJSON_ASSERT(is.Peek() != '\0');
            if (RAPIDJSON_UNLIKELY(!(writeFlags & kWriteValidateEncodingFlag ?
                Transcoder<SourceEncoding, TargetEncoding>::Validate(is, *os_) :
                Transcoder<SourceEncoding, TargetEncoding>::TranscodeUnsafe(is, *os_))))
                return false;
        }
        return true;
    }
    // 在输出流中添加前缀
    void Prefix(Type type) {
        (void)type; // 忽略未使用的参数
        if (RAPIDJSON_LIKELY(level_stack_.GetSize() != 0)) { // 如果值不在根节点
            Level* level = level_stack_.template Top<Level>(); // 获取当前级别
            if (level->valueCount > 0) { // 如果当前级别的值计数大于0
                if (level->inArray) // 如果在数组中
                    os_->Put(','); // 如果不是数组中的第一个元素，则添加逗号
                else  // 在对象中
                    os_->Put((level->valueCount % 2 == 0) ? ',' : ':'); // 如果是偶数个，则添加逗号，否则添加冒号
            }
            if (!level->inArray && level->valueCount % 2 == 0)
                RAPIDJSON_ASSERT(type == kStringType);  // 如果在对象中，则偶数个应该是名称类型
            level->valueCount++; // 值计数加一
        }
        else {
            RAPIDJSON_ASSERT(!hasRoot_);    // 应该只有一个根节点
            hasRoot_ = true; // 设置已经有根节点
        }
    }

    // 如果是顶层值，则刷新值
    bool EndValue(bool ret) {
        if (RAPIDJSON_UNLIKELY(level_stack_.Empty()))   // JSON 文本结束
            Flush(); // 刷新输出流
        return ret; // 返回结果
    }

    OutputStream* os_; // 输出流指针
    internal::Stack<StackAllocator> level_stack_; // 级别堆栈
    int maxDecimalPlaces_; // 最大小数位数
    bool hasRoot_; // 是否有根节点
private:
    // 禁止复制构造函数和赋值运算符
    Writer(const Writer&);
    Writer& operator=(const Writer&);
};

// 针对StringStream的完全特化，以防止内存复制

template<>
inline bool Writer<StringBuffer>::WriteInt(int i) {
    char *buffer = os_->Push(11);  // 分配11个字符的缓冲区
    const char* end = internal::i32toa(i, buffer);  // 将整数i转换为字符串，存储在缓冲区中
    os_->Pop(static_cast<size_t>(11 - (end - buffer)));  // 释放多余的缓冲区空间
    return true;
}

template<>
inline bool Writer<StringBuffer>::WriteUint(unsigned u) {
    char *buffer = os_->Push(10);  // 分配10个字符的缓冲区
    const char* end = internal::u32toa(u, buffer);  // 将无符号整数u转换为字符串，存储在缓冲区中
    os_->Pop(static_cast<size_t>(10 - (end - buffer)));  // 释放多余的缓冲区空间
    return true;
}

template<>
inline bool Writer<StringBuffer>::WriteInt64(int64_t i64) {
    char *buffer = os_->Push(21);  // 分配21个字符的缓冲区
    const char* end = internal::i64toa(i64, buffer);  // 将64位整数i64转换为字符串，存储在缓冲区中
    os_->Pop(static_cast<size_t>(21 - (end - buffer)));  // 释放多余的缓冲区空间
    return true;
}

template<>
inline bool Writer<StringBuffer>::WriteUint64(uint64_t u) {
    char *buffer = os_->Push(20);  // 分配20个字符的缓冲区
    const char* end = internal::u64toa(u, buffer);  // 将64位无符号整数u转换为字符串，存储在缓冲区中
    os_->Pop(static_cast<size_t>(20 - (end - buffer)));  // 释放多余的缓冲区空间
    return true;
}

template<>
inline bool Writer<StringBuffer>::WriteDouble(double d) {
    if (internal::Double(d).IsNanOrInf()) {
        // 注意：只有当(RAPIDJSON_WRITE_DEFAULT_FLAGS & kWriteNanAndInfFlag)时才会进入此代码路径
        if (!(kWriteDefaultFlags & kWriteNanAndInfFlag))
            return false;
        if (internal::Double(d).IsNan()) {
            PutReserve(*os_, 3);  // 预留3个字符的空间
            PutUnsafe(*os_, 'N'); PutUnsafe(*os_, 'a'); PutUnsafe(*os_, 'N');  // 将"Nan"写入缓冲区
            return true;
        }
        if (internal::Double(d).Sign()) {
            PutReserve(*os_, 9);  // 预留9个字符的空间
            PutUnsafe(*os_, '-');  // 将负号写入缓冲区
        }
        else
            PutReserve(*os_, 8);  // 预留8个字符的空间
        PutUnsafe(*os_, 'I'); PutUnsafe(*os_, 'n'); PutUnsafe(*os_, 'f');  // 将"Infinity"写入缓冲区
        PutUnsafe(*os_, 'i'); PutUnsafe(*os_, 'n'); PutUnsafe(*os_, 'i'); PutUnsafe(*os_, 't'); PutUnsafe(*os_, 'y');  // 将"infinity"写入缓冲区
        return true;
    // 从操作系统中获取一个大小为25的字符缓冲区
    char *buffer = os_->Push(25);
    // 使用内部函数将double类型的数字转换为字符串，存储在缓冲区中，并返回字符串的结尾位置
    char* end = internal::dtoa(d, buffer, maxDecimalPlaces_);
    // 释放多余的缓冲区空间，确保只返回有效数据
    os_->Pop(static_cast<size_t>(25 - (end - buffer)));
    // 返回转换成功
    return true;
}
// 如果定义了RAPIDJSON_SSE2或者RAPIDJSON_SSE42，则执行以下模板特化函数
template<>
inline bool Writer<StringBuffer>::ScanWriteUnescapedString(StringStream& is, size_t length) {
    // 如果长度小于16，则返回是否已经读取完毕
    if (length < 16)
        return RAPIDJSON_LIKELY(is.Tell() < length);

    // 如果长度大于等于16且已经读取完毕，则返回false
    if (!RAPIDJSON_LIKELY(is.Tell() < length))
        return false;

    // 获取字符串起始位置和结束位置
    const char* p = is.src_;
    const char* end = is.head_ + length;
    // 将起始位置向上取整到16的倍数
    const char* nextAligned = reinterpret_cast<const char*>((reinterpret_cast<size_t>(p) + 15) & static_cast<size_t>(~15));
    // 将结束位置向下取整到16的倍数
    const char* endAligned = reinterpret_cast<const char*>(reinterpret_cast<size_t>(end) & static_cast<size_t>(~15));
    // 如果下一个对齐位置大于结束位置，则返回true
    if (nextAligned > end)
        return true;

    // 循环处理未对齐部分的字符
    while (p != nextAligned)
        // 如果字符小于0x20或者等于双引号或者等于反斜杠，则返回是否已经读取完毕
        if (*p < 0x20 || *p == '\"' || *p == '\\') {
            is.src_ = p;
            return RAPIDJSON_LIKELY(is.Tell() < length);
        }
        else
            os_->PutUnsafe(*p++);

    // 使用SIMD处理剩余的字符串
    static const char dquote[16] = { '\"', '\"', '\"', '\"', '\"', '\"', '\"', '\"', '\"', '\"', '\"', '\"', '\"', '\"', '\"', '\"' };
    static const char bslash[16] = { '\\', '\\', '\\', '\\', '\\', '\\', '\\', '\\', '\\', '\\', '\\', '\\', '\\', '\\', '\\', '\\' };
    static const char space[16]  = { 0x1F, 0x1F, 0x1F, 0x1F, 0x1F, 0x1F, 0x1F, 0x1F, 0x1F, 0x1F, 0x1F, 0x1F, 0x1F, 0x1F, 0x1F, 0x1F };
    // 加载特定字符到SIMD寄存器
    const __m128i dq = _mm_loadu_si128(reinterpret_cast<const __m128i *>(&dquote[0]));
    const __m128i bs = _mm_loadu_si128(reinterpret_cast<const __m128i *>(&bslash[0]));
    const __m128i sp = _mm_loadu_si128(reinterpret_cast<const __m128i *>(&space[0]));
    // 以16字节对齐的方式遍历数据，每次移动16字节
    for (; p != endAligned; p += 16) {
        // 加载16字节数据到寄存器s
        const __m128i s = _mm_load_si128(reinterpret_cast<const __m128i *>(p));
        // 比较s和dq的每个字节是否相等，结果存储在t1中
        const __m128i t1 = _mm_cmpeq_epi8(s, dq);
        // 比较s和bs的每个字节是否相等，结果存储在t2中
        const __m128i t2 = _mm_cmpeq_epi8(s, bs);
        // 比较s的每个字节是否小于0x20，结果存储在t3中
        const __m128i t3 = _mm_cmpeq_epi8(_mm_max_epu8(s, sp), sp); // s < 0x20 <=> max(s, 0x1F) == 0x1F
        // 将t1、t2、t3的结果合并，存储在x中
        const __m128i x = _mm_or_si128(_mm_or_si128(t1, t2), t3);
        // 将x中每个字节的最高位拼接成一个16位的数，存储在r中
        unsigned short r = static_cast<unsigned short>(_mm_movemask_epi8(x));
        // 如果r不等于0，表示有字符被转义了
        if (RAPIDJSON_UNLIKELY(r != 0)) {   // some of characters is escaped
            // 声明变量len
            SizeType len;
#ifdef _MSC_VER         // 如果是在 MSVC 编译器下
            unsigned long offset;  // 定义一个无符号长整型变量 offset
            _BitScanForward(&offset, r);  // 使用 _BitScanForward 函数找到 r 中第一个置位的位置，存储到 offset 中
            len = offset;  // 将 offset 的值赋给 len
#else
            len = static_cast<SizeType>(__builtin_ffs(r) - 1);  // 如果不是在 MSVC 编译器下，使用 __builtin_ffs 函数找到 r 中第一个置位的位置，然后减去 1 赋给 len
#endif
            char* q = reinterpret_cast<char*>(os_->PushUnsafe(len));  // 将长度为 len 的内存分配给 q
            for (size_t i = 0; i < len; i++)  // 遍历 len 次
                q[i] = p[i];  // 将 p 中的数据复制到 q 中

            p += len;  // 将 p 向后移动 len 个位置
            break;  // 跳出循环
        }
        _mm_storeu_si128(reinterpret_cast<__m128i *>(os_->PushUnsafe(16)), s);  // 将 s 存储到长度为 16 的内存中

    }

    is.src_ = p;  // 将 p 赋给 is.src_
    return RAPIDJSON_LIKELY(is.Tell() < length);  // 返回 is.Tell() < length 的结果
}
#elif defined(RAPIDJSON_NEON)  // 如果定义了 RAPIDJSON_NEON
template<>  // 模板特化
inline bool Writer<StringBuffer>::ScanWriteUnescapedString(StringStream& is, size_t length) {  // 定义函数 ScanWriteUnescapedString，参数为 is 和 length
    if (length < 16)  // 如果 length 小于 16
        return RAPIDJSON_LIKELY(is.Tell() < length);  // 返回 is.Tell() < length 的结果

    if (!RAPIDJSON_LIKELY(is.Tell() < length))  // 如果 is.Tell() >= length
        return false;  // 返回 false

    const char* p = is.src_;  // 定义指针 p，指向 is.src_
    const char* end = is.head_ + length;  // 定义指针 end，指向 is.head_ + length
    const char* nextAligned = reinterpret_cast<const char*>((reinterpret_cast<size_t>(p) + 15) & static_cast<size_t>(~15));  // 计算下一个对齐的地址
    const char* endAligned = reinterpret_cast<const char*>(reinterpret_cast<size_t>(end) & static_cast<size_t>(~15));  // 计算对齐的结束地址
    if (nextAligned > end)  // 如果下一个对齐的地址大于结束地址
        return true;  // 返回 true

    while (p != nextAligned)  // 当 p 不等于下一个对齐的地址时
        if (*p < 0x20 || *p == '\"' || *p == '\\') {  // 如果 *p 小于 0x20 或等于 '"' 或等于 '\\'
            is.src_ = p;  // 将 p 赋给 is.src_
            return RAPIDJSON_LIKELY(is.Tell() < length);  // 返回 is.Tell() < length 的结果
        }
        else
            os_->PutUnsafe(*p++);  // 将 *p 放入 os_

    // The rest of string using SIMD  // 使用 SIMD 处理字符串的剩余部分
    const uint8x16_t s0 = vmovq_n_u8('"');  // 定义 uint8x16_t 类型的变量 s0，存储 '"'
    const uint8x16_t s1 = vmovq_n_u8('\\');  // 定义 uint8x16_t 类型的变量 s1，存储 '\\'
    const uint8x16_t s2 = vmovq_n_u8('\b');  // 定义 uint8x16_t 类型的变量 s2，存储 '\b'
    const uint8x16_t s3 = vmovq_n_u8(32);  // 定义 uint8x16_t 类型的变量 s3，存储 32
    // 以16字节对齐方式遍历数据，每次增加16字节
    for (; p != endAligned; p += 16) {
        // 从指针p处加载16字节数据到寄存器s
        const uint8x16_t s = vld1q_u8(reinterpret_cast<const uint8_t *>(p));
        // 逐个比较s和s0、s1、s2、s3的相等性，将结果存储到x中
        uint8x16_t x = vceqq_u8(s, s0);
        x = vorrq_u8(x, vceqq_u8(s, s1));
        x = vorrq_u8(x, vceqq_u8(s, s2));
        x = vorrq_u8(x, vcltq_u8(s, s3));

        // 对x进行64位反转
        x = vrev64q_u8(x);                     // Rev in 64
        // 从x中提取低64位和高64位数据
        uint64_t low = vgetq_lane_u64(vreinterpretq_u64_u8(x), 0);   // extract
        uint64_t high = vgetq_lane_u64(vreinterpretq_u64_u8(x), 1);  // extract

        // 初始化长度和转义标志
        SizeType len = 0;
        bool escaped = false;
        // 判断是否需要转义
        if (low == 0) {
            if (high != 0) {
                uint32_t lz = internal::clzll(high);
                len = 8 + (lz >> 3);
                escaped = true;
            }
        } else {
            uint32_t lz = internal::clzll(low);
            len = lz >> 3;
            escaped = true;
        }
        // 如果需要转义
        if (RAPIDJSON_UNLIKELY(escaped)) {   // some of characters is escaped
            // 将p中的数据拷贝到os_中，并更新p的位置
            char* q = reinterpret_cast<char*>(os_->PushUnsafe(len));
            for (size_t i = 0; i < len; i++)
                q[i] = p[i];

            p += len;
            break;
        }
        // 将s中的数据存储到os_中，并更新p的位置
        vst1q_u8(reinterpret_cast<uint8_t *>(os_->PushUnsafe(16)), s);
    }

    // 更新is.src_的值为p
    is.src_ = p;
    // 返回is.Tell() < length的结果
    return RAPIDJSON_LIKELY(is.Tell() < length);
}
#endif // RAPIDJSON_NEON

RAPIDJSON_NAMESPACE_END

#if defined(_MSC_VER) || defined(__clang__)
RAPIDJSON_DIAG_POP
#endif

#endif // RAPIDJSON_RAPIDJSON_H_
```