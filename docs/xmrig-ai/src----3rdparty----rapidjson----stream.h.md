# `xmrig\src\3rdparty\rapidjson\stream.h`

```cpp
// 包含版权声明和许可证信息
// 定义了 rapidjson 命名空间
// 包含了 encodings.h 文件

#ifndef RAPIDJSON_STREAM_H_
#define RAPIDJSON_STREAM_H_

#include "encodings.h"

RAPIDJSON_NAMESPACE_BEGIN

///////////////////////////////////////////////////////////////////////////////
//  Stream

/*! \class rapidjson::Stream
    \brief 用于读取和写入字符的概念。

    对于只读流，不需要实现 PutBegin()、Put()、Flush() 和 PutEnd()。

    对于只写流，只需要实现 Put() 和 Flush()。

\code
concept Stream {
    typename Ch;    //!< 流的字符类型。

    //! 从流中读取当前字符，但不移动读取光标。
    Ch Peek() const;

    //! 从流中读取当前字符，并将读取光标移动到下一个字符。
    Ch Take();

    //! 获取当前读取光标的位置。
    //! \return 从开头读取的字符数。
    size_t Tell();

    //! 在当前读取指针处开始写操作。
    //! \return 开始写指针。
    Ch* PutBegin();

    //! 写入一个字符。
    void Put(Ch c);

    //! 刷新缓冲区。
    void Flush();

    //! 结束写操作。
    //! \param begin PutBegin() 返回的开始写指针。
    //! \return 写入的字符数。
    size_t PutEnd(Ch* begin);
}
\endcode
*/
//! Provides additional information for stream.
/*!
    By using traits pattern, this type provides a default configuration for stream.
    For custom stream, this type can be specialized for other configuration.
    See TEST(Reader, CustomStringStream) in readertest.cpp for example.
*/
template<typename Stream>
struct StreamTraits {
    //! Whether to make local copy of stream for optimization during parsing.
    /*!
        By default, for safety, streams do not use local copy optimization.
        Stream that can be copied fast should specialize this, like StreamTraits<StringStream>.
    */
    enum { copyOptimization = 0 };
};

//! Reserve n characters for writing to a stream.
template<typename Stream>
inline void PutReserve(Stream& stream, size_t count) {
    (void)stream;
    (void)count;
}

//! Write character to a stream, presuming buffer is reserved.
template<typename Stream>
inline void PutUnsafe(Stream& stream, typename Stream::Ch c) {
    stream.Put(c);
}

//! Put N copies of a character to a stream.
template<typename Stream, typename Ch>
inline void PutN(Stream& stream, Ch c, size_t n) {
    PutReserve(stream, n);
    for (size_t i = 0; i < n; i++)
        PutUnsafe(stream, c);
}

///////////////////////////////////////////////////////////////////////////////
// GenericStreamWrapper

//! A Stream Wrapper
/*! \tThis string stream is a wrapper for any stream by just forwarding any
    \treceived message to the origin stream.
    \note implements Stream concept
*/

#if defined(_MSC_VER) && _MSC_VER <= 1800
RAPIDJSON_DIAG_PUSH
RAPIDJSON_DIAG_OFF(4702)  // unreachable code
RAPIDJSON_DIAG_OFF(4512)  // assignment operator could not be generated
#endif

template <typename InputStream, typename Encoding = UTF8<> >
class GenericStreamWrapper {
public:
    typedef typename Encoding::Ch Ch;
    GenericStreamWrapper(InputStream& is): is_(is) {}

    Ch Peek() const { return is_.Peek(); } // 返回当前字符但不移动读取位置
    Ch Take() { return is_.Take(); } // 返回当前字符并移动读取位置
    size_t Tell() { return is_.Tell(); } // 返回当前读取位置
    # 返回指向输入流的指针
    Ch* PutBegin() { return is_.PutBegin(); }
    # 将字符放入输入流
    void Put(Ch ch) { is_.Put(ch); }
    # 刷新输入流
    void Flush() { is_.Flush(); }
    # 结束输入流，返回字符指针
    size_t PutEnd(Ch* ch) { return is_.PutEnd(ch); }

    # 获取输入流中的前4个字符
    const Ch* Peek4() const { return is_.Peek4(); }

    # 获取输入流的编码类型
    UTFType GetType() const { return is_.GetType(); }
    # 检查输入流是否有BOM（字节顺序标记）
    bool HasBOM() const { return is_.HasBOM(); }
protected:
    InputStream& is_;
};

#if defined(_MSC_VER) && _MSC_VER <= 1800
RAPIDJSON_DIAG_POP
#endif

///////////////////////////////////////////////////////////////////////////////
// StringStream

//! Read-only string stream.
/*! \note implements Stream concept
*/
template <typename Encoding>
struct GenericStringStream {
    typedef typename Encoding::Ch Ch;

    GenericStringStream(const Ch *src) : src_(src), head_(src) {}  // 用给定的字符串初始化流的起始位置和头部位置

    Ch Peek() const { return *src_; }  // 返回当前位置的字符
    Ch Take() { return *src_++; }  // 返回当前位置的字符，并将位置后移一位
    size_t Tell() const { return static_cast<size_t>(src_ - head_); }  // 返回当前位置相对于头部位置的偏移量

    Ch* PutBegin() { RAPIDJSON_ASSERT(false); return 0; }  // 开始写入位置，但此流为只读流，因此不支持写入操作
    void Put(Ch) { RAPIDJSON_ASSERT(false); }  // 写入字符，但此流为只读流，因此不支持写入操作
    void Flush() { RAPIDJSON_ASSERT(false); }  // 刷新操作，但此流为只读流，因此不支持刷新操作
    size_t PutEnd(Ch*) { RAPIDJSON_ASSERT(false); return 0; }  // 结束写入操作，但此流为只读流，因此不支持写入操作

    const Ch* src_;     //!< Current read position.  // 当前读取位置
    const Ch* head_;    //!< Original head of the string.  // 字符串的原始头部位置
};

template <typename Encoding>
struct StreamTraits<GenericStringStream<Encoding> > {
    enum { copyOptimization = 1 };  // 流特性，支持复制优化
};

//! String stream with UTF8 encoding.
typedef GenericStringStream<UTF8<> > StringStream;

///////////////////////////////////////////////////////////////////////////////
// InsituStringStream

//! A read-write string stream.
/*! This string stream is particularly designed for in-situ parsing.
    \note implements Stream concept
*/
template <typename Encoding>
struct GenericInsituStringStream {
    typedef typename Encoding::Ch Ch;

    GenericInsituStringStream(Ch *src) : src_(src), dst_(0), head_(src) {}  // 用给定的字符串初始化流的起始位置、写入位置和头部位置

    // Read
    Ch Peek() { return *src_; }  // 返回当前位置的字符
    Ch Take() { return *src_++; }  // 返回当前位置的字符，并将位置后移一位
    size_t Tell() { return static_cast<size_t>(src_ - head_); }  // 返回当前位置相对于头部位置的偏移量

    // Write
    void Put(Ch c) { RAPIDJSON_ASSERT(dst_ != 0); *dst_++ = c; }  // 写入字符到写入位置，并将写入位置后移一位

    Ch* PutBegin() { return dst_ = src_; }  // 返回写入位置，并将写入位置设置为起始位置
    size_t PutEnd(Ch* begin) { return static_cast<size_t>(dst_ - begin); }  // 返回写入的字符数
    void Flush() {}  // 刷新操作

    Ch* Push(size_t count) { Ch* begin = dst_; dst_ += count; return begin; }  // 将写入位置后移指定的字符数，并返回移动前的写入位置
    # 定义一个函数 Pop，用于将 dst_ 指针向前移动指定的大小
    void Pop(size_t count) { dst_ -= count; }

    # 定义三个指针变量 src_、dst_、head_
    Ch* src_;
    Ch* dst_;
    Ch* head_;
// 结构体模板，用于定义 GenericInsituStringStream<Encoding> 的流特性
template <typename Encoding>
struct StreamTraits<GenericInsituStringStream<Encoding> > {
    // 设置复制优化标志为1
    enum { copyOptimization = 1 };
};

// 定义使用 UTF8 编码的 InsituStringStream 类型
typedef GenericInsituStringStream<UTF8<> > InsituStringStream;

// 结束 RapidJSON 命名空间
RAPIDJSON_NAMESPACE_END

// 结束条件编译，关闭流文件
#endif // RAPIDJSON_STREAM_H_
```