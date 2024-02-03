# `xmrig\src\3rdparty\rapidjson\filewritestream.h`

```cpp
#ifndef RAPIDJSON_FILEWRITESTREAM_H_
#define RAPIDJSON_FILEWRITESTREAM_H_

#include "stream.h"
#include <cstdio>

#ifdef __clang__
RAPIDJSON_DIAG_PUSH
RAPIDJSON_DIAG_OFF(unreachable-code)
#endif

RAPIDJSON_NAMESPACE_BEGIN

//! Wrapper of C file stream for output using fwrite().
/*!
    \note implements Stream concept
*/
class FileWriteStream {
public:
    typedef char Ch;    //!< Character type. Only support char.

    // 构造函数，初始化文件指针、缓冲区和当前位置
    FileWriteStream(std::FILE* fp, char* buffer, size_t bufferSize) : fp_(fp), buffer_(buffer), bufferEnd_(buffer + bufferSize), current_(buffer_) {
        RAPIDJSON_ASSERT(fp_ != 0);
    }

    // 写入单个字符到缓冲区
    void Put(char c) {
        if (current_ >= bufferEnd_)
            Flush();

        *current_++ = c;
    }

    // 写入指定数量的字符到缓冲区
    void PutN(char c, size_t n) {
        size_t avail = static_cast<size_t>(bufferEnd_ - current_);
        while (n > avail) {
            std::memset(current_, c, avail);
            current_ += avail;
            Flush();
            n -= avail;
            avail = static_cast<size_t>(bufferEnd_ - current_);
        }

        if (n > 0) {
            std::memset(current_, c, n);
            current_ += n;
        }
    }
    // 刷新缓冲区
    void Flush() {
        // 如果当前指针不等于缓冲区指针
        if (current_ != buffer_) {
            // 将缓冲区的内容写入文件流中
            size_t result = std::fwrite(buffer_, 1, static_cast<size_t>(current_ - buffer_), fp_);
            // 如果写入的字节数小于缓冲区的大小
            if (result < static_cast<size_t>(current_ - buffer_)) {
                // 忽略故意忽略的失败，以避免 warn_unused_result 构建错误
                // 为了避免 warn_unused_result 构建错误而添加
            }
            // 重置当前指针为缓冲区指针
            current_ = buffer_;
        }
    }

    // 未实现的方法
    char Peek() const { RAPIDJSON_ASSERT(false); return 0; }
    char Take() { RAPIDJSON_ASSERT(false); return 0; }
    size_t Tell() const { RAPIDJSON_ASSERT(false); return 0; }
    char* PutBegin() { RAPIDJSON_ASSERT(false); return 0; }
    size_t PutEnd(char*) { RAPIDJSON_ASSERT(false); return 0; }
private:
    // 禁止复制构造函数和赋值运算符。
    FileWriteStream(const FileWriteStream&);
    FileWriteStream& operator=(const FileWriteStream&);

    std::FILE* fp_; // 文件指针
    char *buffer_; // 缓冲区指针
    char *bufferEnd_; // 缓冲区结束指针
    char *current_; // 当前指针
};

//! 为了提高性能，使用memset()实现PutN()的特殊版本。
template<>
inline void PutN(FileWriteStream& stream, char c, size_t n) {
    stream.PutN(c, n);
}

RAPIDJSON_NAMESPACE_END

#ifdef __clang__
RAPIDJSON_DIAG_POP
#endif

#endif // RAPIDJSON_FILESTREAM_H_
```