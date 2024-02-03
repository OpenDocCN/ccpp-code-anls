# `xmrig\src\3rdparty\rapidjson\encodedstream.h`

```cpp
#ifndef RAPIDJSON_ENCODEDSTREAM_H_
#define RAPIDJSON_ENCODEDSTREAM_H_

#include "stream.h"
#include "memorystream.h"

#ifdef __GNUC__
RAPIDJSON_DIAG_PUSH
RAPIDJSON_DIAG_OFF(effc++)
#endif

#ifdef __clang__
RAPIDJSON_DIAG_PUSH
RAPIDJSON_DIAG_OFF(padded)
#endif

RAPIDJSON_NAMESPACE_BEGIN

//! Input byte stream wrapper with a statically bound encoding.
/*!
    \tparam Encoding The interpretation of encoding of the stream. Either UTF8, UTF16LE, UTF16BE, UTF32LE, UTF32BE.
    \tparam InputByteStream Type of input byte stream. For example, FileReadStream.
*/
template <typename Encoding, typename InputByteStream>
class EncodedInputStream {
    RAPIDJSON_STATIC_ASSERT(sizeof(typename InputByteStream::Ch) == 1);
public:
    typedef typename Encoding::Ch Ch;

    // 构造函数，初始化输入字节流和当前字符
    EncodedInputStream(InputByteStream& is) : is_(is) {
        current_ = Encoding::TakeBOM(is_);
    }

    // 返回当前字符
    Ch Peek() const { return current_; }
    // 取出当前字符并返回，更新当前字符
    Ch Take() { Ch c = current_; current_ = Encoding::Take(is_); return c; }
    // 返回当前位置
    size_t Tell() const { return is_.Tell(); }

    // 以下方法未实现
    void Put(Ch) { RAPIDJSON_ASSERT(false); }
    void Flush() { RAPIDJSON_ASSERT(false); }
    Ch* PutBegin() { RAPIDJSON_ASSERT(false); return 0; }
    size_t PutEnd(Ch*) { RAPIDJSON_ASSERT(false); return 0; }

private:
    # 复制构造函数，禁止使用
    EncodedInputStream(const EncodedInputStream&);
    
    # 赋值运算符重载，禁止使用
    EncodedInputStream& operator=(const EncodedInputStream&);
    
    # 输入字节流的引用
    InputByteStream& is_;
    
    # 当前字符
    Ch current_;
};

//! 专门为 UTF8 MemoryStream 定制的类
template <>
class EncodedInputStream<UTF8<>, MemoryStream> {
public:
    typedef UTF8<>::Ch Ch;

    // 构造函数，初始化输入流
    EncodedInputStream(MemoryStream& is) : is_(is) {
        // 如果流的下一个字节是 0xEFu，则跳过
        if (static_cast<unsigned char>(is_.Peek()) == 0xEFu) is_.Take();
        // 如果流的下一个字节是 0xBBu，则跳过
        if (static_cast<unsigned char>(is_.Peek()) == 0xBBu) is_.Take();
        // 如果流的下一个字节是 0xBFu，则跳过
        if (static_cast<unsigned char>(is_.Peek()) == 0xBFu) is_.Take();
    }
    // 返回下一个字符而不移动读取位置
    Ch Peek() const { return is_.Peek(); }
    // 返回下一个字符并移动读取位置
    Ch Take() { return is_.Take(); }
    // 返回当前读取位置
    size_t Tell() const { return is_.Tell(); }

    // 未实现的方法
    void Put(Ch) {}
    void Flush() {}
    Ch* PutBegin() { return 0; }
    size_t PutEnd(Ch*) { return 0; }

    // 输入流对象
    MemoryStream& is_;

private:
    // 复制构造函数和赋值运算符重载被私有化
    EncodedInputStream(const EncodedInputStream&);
    EncodedInputStream& operator=(const EncodedInputStream&);
};

//! 输出字节流包装器，具有静态绑定的编码
/*!
    \tparam Encoding 流的编码解释，可以是 UTF8, UTF16LE, UTF16BE, UTF32LE, UTF32BE 中的一种
    \tparam OutputByteStream 输入字节流的类型，例如 FileWriteStream
*/
template <typename Encoding, typename OutputByteStream>
class EncodedOutputStream {
    RAPIDJSON_STATIC_ASSERT(sizeof(typename OutputByteStream::Ch) == 1);
public:
    typedef typename Encoding::Ch Ch;

    // 构造函数，初始化输出流
    EncodedOutputStream(OutputByteStream& os, bool putBOM = true) : os_(os) {
        // 如果需要，写入字节顺序标记
        if (putBOM)
            Encoding::PutBOM(os_);
    }

    // 写入字符到输出流
    void Put(Ch c) { Encoding::Put(os_, c);  }
    // 刷新输出流
    void Flush() { os_.Flush(); }

    // 未实现的方法
    Ch Peek() const { RAPIDJSON_ASSERT(false); return 0;}
    Ch Take() { RAPIDJSON_ASSERT(false); return 0;}
    size_t Tell() const { RAPIDJSON_ASSERT(false);  return 0; }
    Ch* PutBegin() { RAPIDJSON_ASSERT(false); return 0; }
    size_t PutEnd(Ch*) { RAPIDJSON_ASSERT(false); return 0; }

private:
    // 复制构造函数和赋值运算符重载被私有化
    EncodedOutputStream(const EncodedOutputStream&);
    EncodedOutputStream& operator=(const EncodedOutputStream&);

    // 输出流对象
    OutputByteStream& os_;
};

#define RAPIDJSON_ENCODINGS_FUNC(x) UTF8<Ch>::x, UTF16LE<Ch>::x, UTF16BE<Ch>::x, UTF32LE<Ch>::x, UTF32BE<Ch>::x

//! Input stream wrapper with dynamically bound encoding and automatic encoding detection.
/*!
    \tparam CharType Type of character for reading.
    \tparam InputByteStream type of input byte stream to be wrapped.
*/
template <typename CharType, typename InputByteStream>
class AutoUTFInputStream {
    RAPIDJSON_STATIC_ASSERT(sizeof(typename InputByteStream::Ch) == 1);
public:
    typedef CharType Ch;

    //! Constructor.
    /*!
        \param is input stream to be wrapped.
        \param type UTF encoding type if it is not detected from the stream.
    */
    AutoUTFInputStream(InputByteStream& is, UTFType type = kUTF8) : is_(&is), type_(type), hasBOM_(false) {
        RAPIDJSON_ASSERT(type >= kUTF8 && type <= kUTF32BE);
        DetectType();
        static const TakeFunc f[] = { RAPIDJSON_ENCODINGS_FUNC(Take) };
        takeFunc_ = f[type_];
        current_ = takeFunc_(*is_);
    }

    UTFType GetType() const { return type_; }
    bool HasBOM() const { return hasBOM_; }

    Ch Peek() const { return current_; }
    Ch Take() { Ch c = current_; current_ = takeFunc_(*is_); return c; }
    size_t Tell() const { return is_->Tell(); }

    // Not implemented
    void Put(Ch) { RAPIDJSON_ASSERT(false); }
    void Flush() { RAPIDJSON_ASSERT(false); }
    Ch* PutBegin() { RAPIDJSON_ASSERT(false); return 0; }
    size_t PutEnd(Ch*) { RAPIDJSON_ASSERT(false); return 0; }

private:
    AutoUTFInputStream(const AutoUTFInputStream&);
    AutoUTFInputStream& operator=(const AutoUTFInputStream&);

    // Detect encoding type with BOM or RFC 4627
    }

    typedef Ch (*TakeFunc)(InputByteStream& is);
    InputByteStream* is_;
    UTFType type_;
    Ch current_;
    TakeFunc takeFunc_;
    bool hasBOM_;
};

//! Output stream wrapper with dynamically bound encoding and automatic encoding detection.
/*!
    \tparam CharType Type of character for writing.
    # 参数说明：OutputByteStream是要封装的输出字节流的类型
*/
// 定义一个模板类 AutoUTFOutputStream，用于将不同编码类型的字符流输出到指定的输出流中
template <typename CharType, typename OutputByteStream>
class AutoUTFOutputStream {
    // 静态断言，检查输出流的字符类型是否为 1 字节
    RAPIDJSON_STATIC_ASSERT(sizeof(typename OutputByteStream::Ch) == 1);
public:
    typedef CharType Ch;

    //! 构造函数
    /*!
        \param os 要包装的输出流
        \param type UTF 编码类型
        \param putBOM 是否在流的开头写入 BOM
    */
    AutoUTFOutputStream(OutputByteStream& os, UTFType type, bool putBOM) : os_(&os), type_(type) {
        RAPIDJSON_ASSERT(type >= kUTF8 && type <= kUTF32BE);

        // 运行时检查字符类型的大小是否足够，只在断言中执行检查
        if (type_ == kUTF16LE || type_ == kUTF16BE) RAPIDJSON_ASSERT(sizeof(Ch) >= 2);
        if (type_ == kUTF32LE || type_ == kUTF32BE) RAPIDJSON_ASSERT(sizeof(Ch) >= 4);

        // 定义一个函数指针数组，根据编码类型选择对应的函数
        static const PutFunc f[] = { RAPIDJSON_ENCODINGS_FUNC(Put) };
        putFunc_ = f[type_];

        // 如果需要写入 BOM，则调用 PutBOM 函数
        if (putBOM)
            PutBOM();
    }

    // 获取编码类型
    UTFType GetType() const { return type_; }

    // 写入单个字符
    void Put(Ch c) { putFunc_(*os_, c); }
    // 刷新输出流
    void Flush() { os_->Flush(); }

    // 未实现的函数
    Ch Peek() const { RAPIDJSON_ASSERT(false); return 0;}
    Ch Take() { RAPIDJSON_ASSERT(false); return 0;}
    size_t Tell() const { RAPIDJSON_ASSERT(false); return 0; }
    Ch* PutBegin() { RAPIDJSON_ASSERT(false); return 0; }
    size_t PutEnd(Ch*) { RAPIDJSON_ASSERT(false); return 0; }

private:
    AutoUTFOutputStream(const AutoUTFOutputStream&);
    AutoUTFOutputStream& operator=(const AutoUTFOutputStream&);

    // 写入 BOM
    void PutBOM() {
        typedef void (*PutBOMFunc)(OutputByteStream&);
        static const PutBOMFunc f[] = { RAPIDJSON_ENCODINGS_FUNC(PutBOM) };
        f[type_](*os_);
    }

    // 定义一个函数指针类型
    typedef void (*PutFunc)(OutputByteStream&, Ch);

    OutputByteStream* os_;
    UTFType type_;
    PutFunc putFunc_;
};

// 取消定义 RAPIDJSON_ENCODINGS_FUNC
#undef RAPIDJSON_ENCODINGS_FUNC

// 结束 RapidJSON 命名空间
RAPIDJSON_NAMESPACE_END

// 恢复之前的诊断状态
#ifdef __clang__
RAPIDJSON_DIAG_POP
#endif

#ifdef __GNUC__
// 弹出最近的诊断状态
RAPIDJSON_DIAG_POP
#endif

#endif // RAPIDJSON_FILESTREAM_H_
```