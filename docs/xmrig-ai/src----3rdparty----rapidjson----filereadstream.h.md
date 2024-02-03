# `xmrig\src\3rdparty\rapidjson\filereadstream.h`

```cpp
#ifndef RAPIDJSON_FILEREADSTREAM_H_
#define RAPIDJSON_FILEREADSTREAM_H_

#include "stream.h"
#include <cstdio>

#ifdef __clang__
RAPIDJSON_DIAG_PUSH
RAPIDJSON_DIAG_OFF(padded)
RAPIDJSON_DIAG_OFF(unreachable-code)
RAPIDJSON_DIAG_OFF(missing-noreturn)
#endif

RAPIDJSON_NAMESPACE_BEGIN

//! File byte stream for input using fread().
/*!
    \note implements Stream concept
*/
class FileReadStream {
public:
    typedef char Ch;    //!< Character type (byte).

    //! Constructor.
    /*!
        \param fp File pointer opened for read.
        \param buffer user-supplied buffer.
        \param bufferSize size of buffer in bytes. Must >=4 bytes.
    */
    // 构造函数，初始化文件指针、缓冲区和相关变量
    FileReadStream(std::FILE* fp, char* buffer, size_t bufferSize) : fp_(fp), buffer_(buffer), bufferSize_(bufferSize), bufferLast_(0), current_(buffer_), readCount_(0), count_(0), eof_(false) {
        RAPIDJSON_ASSERT(fp_ != 0);  // 断言文件指针不为空
        RAPIDJSON_ASSERT(bufferSize >= 4);  // 断言缓冲区大小至少为4字节
        Read();  // 读取文件内容到缓冲区
    }

    // 返回当前字符
    Ch Peek() const { return *current_; }
    // 返回当前字符并移动到下一个字符
    Ch Take() { Ch c = *current_; Read(); return c; }
    // 返回当前读取位置的偏移量
    size_t Tell() const { return count_ + static_cast<size_t>(current_ - buffer_); }

    // 以下方法未实现
    void Put(Ch) { RAPIDJSON_ASSERT(false); }
    void Flush() { RAPIDJSON_ASSERT(false); }
    Ch* PutBegin() { RAPIDJSON_ASSERT(false); return 0; }
    # 定义一个返回类型为 size_t，参数为 Ch* 的 PutEnd 函数，该函数断言为假，返回 0
    size_t PutEnd(Ch*) { RAPIDJSON_ASSERT(false); return 0; }

    # 仅用于编码检测
    # 返回当前指针后的 4 个字符，如果当前指针加上 4 减去 eof_ 的值小于等于 bufferLast_，则返回当前指针，否则返回 0
    const Ch* Peek4() const {
        return (current_ + 4 - !eof_ <= bufferLast_) ? current_ : 0;
    }
// 读取文件内容的私有方法
private:
    void Read() {
        // 如果当前指针小于缓冲区最后一个位置，则将当前指针向后移动一位
        if (current_ < bufferLast_)
            ++current_;
        // 如果不是文件结尾
        else if (!eof_) {
            // 更新已读取字符数
            count_ += readCount_;
            // 从文件中读取数据到缓冲区
            readCount_ = std::fread(buffer_, 1, bufferSize_, fp_);
            // 更新缓冲区最后一个位置的指针
            bufferLast_ = buffer_ + readCount_ - 1;
            // 将当前指针指向缓冲区起始位置
            current_ = buffer_;

            // 如果读取的字符数小于缓冲区大小
            if (readCount_ < bufferSize_) {
                // 在缓冲区末尾添加字符串结束符
                buffer_[readCount_] = '\0';
                // 更新缓冲区最后一个位置的指针
                ++bufferLast_;
                // 标记已到达文件结尾
                eof_ = true;
            }
        }
    }

    // 文件指针
    std::FILE* fp_;
    // 缓冲区
    Ch *buffer_;
    // 缓冲区大小
    size_t bufferSize_;
    // 缓冲区最后一个位置的指针
    Ch *bufferLast_;
    // 当前指针
    Ch *current_;
    // 已读取字符数
    size_t readCount_;
    // 已读取字符总数
    size_t count_;  //!< Number of characters read
    // 是否到达文件结尾
    bool eof_;
};

// 命名空间结束
RAPIDJSON_NAMESPACE_END

// 如果是 clang 编译器，则恢复警告设置
#ifdef __clang__
RAPIDJSON_DIAG_POP
#endif

// 结束文件流头文件定义
#endif // RAPIDJSON_FILESTREAM_H_
```