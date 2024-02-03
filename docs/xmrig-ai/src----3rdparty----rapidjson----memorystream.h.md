# `xmrig\src\3rdparty\rapidjson\memorystream.h`

```cpp
#ifndef RAPIDJSON_MEMORYSTREAM_H_  // 如果未定义 RAPIDJSON_MEMORYSTREAM_H_，则包含以下代码
#define RAPIDJSON_MEMORYSTREAM_H_

#include "stream.h"  // 包含 stream.h 文件

#ifdef __clang__  // 如果是 clang 编译器
RAPIDJSON_DIAG_PUSH  // 忽略 clang 编译器的警告
RAPIDJSON_DIAG_OFF(unreachable-code)  // 关闭 unreachable-code 警告
RAPIDJSON_DIAG_OFF(missing-noreturn)  // 关闭 missing-noreturn 警告
#endif

RAPIDJSON_NAMESPACE_BEGIN  // 开始命名空间 RAPIDJSON_NAMESPACE_BEGIN

//! 表示内存中的输入字节流
/*!
    该类主要用于被 EncodedInputStream 或 AutoUTFInputStream 包装。

    它类似于 FileReadBuffer，但源是内存缓冲区而不是文件。

    MemoryStream 和 StringStream 之间的区别：
    1. StringStream 有编码，而 MemoryStream 是字节流。
    2. MemoryStream 需要源缓冲区的大小，而缓冲区不需要以空字符结尾。StringStream 假定源是以空字符结尾的字符串。
    3. MemoryStream 支持 Peek4() 用于编码检测。StringStream 指定了编码，因此不应该有 Peek4()。
    \note 实现了 Stream 概念
*/
struct MemoryStream {
    typedef char Ch;  // 字节

    MemoryStream(const Ch *src, size_t size) : src_(src), begin_(src), end_(src + size), size_(size) {}  // 构造函数，初始化源、起始、结束和大小

    Ch Peek() const { return RAPIDJSON_UNLIKELY(src_ == end_) ? '\0' : *src_; }  // 返回当前位置的字符，如果已经到达结束位置，则返回空字符
    Ch Take() { return RAPIDJSON_UNLIKELY(src_ == end_) ? '\0' : *src_++; }  // 返回当前位置的字符，并将位置移动到下一个字符
    size_t Tell() const { return static_cast<size_t>(src_ - begin_); }  // 返回当前位置相对于起始位置的偏移量

... (以下部分代码未提供，需要继续添加注释)
    // 返回空指针，并断言程序错误
    Ch* PutBegin() { RAPIDJSON_ASSERT(false); return 0; }
    // 断言程序错误
    void Put(Ch) { RAPIDJSON_ASSERT(false); }
    // 断言程序错误
    void Flush() { RAPIDJSON_ASSERT(false); }
    // 返回 0，并断言程序错误
    size_t PutEnd(Ch*) { RAPIDJSON_ASSERT(false); return 0; }

    // 仅用于编码检测
    // 如果当前读取位置加上4小于等于流的大小，则返回当前位置指针，否则返回空指针
    const Ch* Peek4() const {
        return Tell() + 4 <= size_ ? src_ : 0;
    }

    const Ch* src_;     //!< 当前读取位置
    const Ch* begin_;   //!< 字符串的原始头部
    const Ch* end_;     //!< 流的末尾
    size_t size_;       //!< 流的大小
};

RAPIDJSON_NAMESPACE_END

#ifdef __clang__
RAPIDJSON_DIAG_POP
#endif

#endif // RAPIDJSON_MEMORYBUFFER_H_
```