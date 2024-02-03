# `xmrig\src\3rdparty\rapidjson\istreamwrapper.h`

```cpp
#ifndef RAPIDJSON_ISTREAMWRAPPER_H_
#define RAPIDJSON_ISTREAMWRAPPER_H_

#include "stream.h"
#include <iosfwd>
#include <ios>

#ifdef __clang__
RAPIDJSON_DIAG_PUSH
RAPIDJSON_DIAG_OFF(padded)
#elif defined(_MSC_VER)
RAPIDJSON_DIAG_PUSH
RAPIDJSON_DIAG_OFF(4351) // new behavior: elements of array 'array' will be default initialized
#endif

RAPIDJSON_NAMESPACE_BEGIN

//! Wrapper of \c std::basic_istream into RapidJSON's Stream concept.
/*!
    The classes can be wrapped including but not limited to:

    - \c std::istringstream
    - \c std::stringstream
    - \c std::wistringstream
    - \c std::wstringstream
    - \c std::ifstream
    - \c std::fstream
    - \c std::wifstream
    - \c std::wfstream

    \tparam StreamType Class derived from \c std::basic_istream.
*/
template <typename StreamType>
class BasicIStreamWrapper {
public:
    typedef typename StreamType::char_type Ch;

    //! Constructor.
    /*!
        \param stream stream opened for read.
    */
    BasicIStreamWrapper(StreamType &stream) : stream_(stream), buffer_(peekBuffer_), bufferSize_(4), bufferLast_(0), current_(buffer_), readCount_(0), count_(0), eof_(false) {
        Read();
    }

    //! Constructor.
    /*!
        \param stream stream opened for read.  // 为读取而打开的流
        \param buffer user-supplied buffer.  // 用户提供的缓冲区
        \param bufferSize size of buffer in bytes. Must >=4 bytes.  // 缓冲区的大小（以字节为单位）。必须大于等于4个字节
    */
    BasicIStreamWrapper(StreamType &stream, char* buffer, size_t bufferSize) : stream_(stream), buffer_(buffer), bufferSize_(bufferSize), bufferLast_(0), current_(buffer_), readCount_(0), count_(0), eof_(false) {
        RAPIDJSON_ASSERT(bufferSize >= 4);  // 断言缓冲区大小必须大于等于4
        Read();  // 调用Read函数
    }

    Ch Peek() const { return *current_; }  // 返回当前字符
    Ch Take() { Ch c = *current_; Read(); return c; }  // 返回当前字符并移动到下一个字符
    size_t Tell() const { return count_ + static_cast<size_t>(current_ - buffer_); }  // 返回当前位置

    // Not implemented
    void Put(Ch) { RAPIDJSON_ASSERT(false); }  // 未实现的函数
    void Flush() { RAPIDJSON_ASSERT(false); }  // 未实现的函数
    Ch* PutBegin() { RAPIDJSON_ASSERT(false); return 0; }  // 未实现的函数
    size_t PutEnd(Ch*) { RAPIDJSON_ASSERT(false); return 0; }  // 未实现的函数

    // For encoding detection only.
    const Ch* Peek4() const {
        return (current_ + 4 - !eof_ <= bufferLast_) ? current_ : 0;  // 仅用于编码检测
    }
// 默认构造函数，私有化，禁止调用
private:
    BasicIStreamWrapper();

// 拷贝构造函数，私有化，禁止调用
    BasicIStreamWrapper(const BasicIStreamWrapper&);

// 赋值运算符重载，私有化，禁止调用
    BasicIStreamWrapper& operator=(const BasicIStreamWrapper&);

// 读取函数
    void Read() {
        // 如果当前指针小于缓冲区最后一个位置，则将当前指针向后移动一位
        if (current_ < bufferLast_)
            ++current_;
        // 如果不是文件末尾
        else if (!eof_) {
            // 更新已读取字符数
            count_ += readCount_;
            // 重置读取计数
            readCount_ = bufferSize_;
            // 更新缓冲区最后一个位置
            bufferLast_ = buffer_ + readCount_ - 1;
            // 重置当前指针
            current_ = buffer_;

            // 从流中读取数据到缓冲区
            if (!stream_.read(buffer_, static_cast<std::streamsize>(bufferSize_))) {
                // 获取实际读取的字符数
                readCount_ = static_cast<size_t>(stream_.gcount());
                // 在缓冲区最后一个位置添加结束符
                *(bufferLast_ = buffer_ + readCount_) = '\0';
                // 标记已到达文件末尾
                eof_ = true;
            }
        }
    }

// 流对象的引用
    StreamType &stream_;
// 预读缓冲区
    Ch peekBuffer_[4], *buffer_;
// 缓冲区大小
    size_t bufferSize_;
// 缓冲区最后一个位置
    Ch *bufferLast_;
// 当前指针
    Ch *current_;
// 实际读取的字符数
    size_t readCount_;
// 已读取字符数
    size_t count_;  //!< Number of characters read
// 是否已到达文件末尾
    bool eof_;
};

// 定义基于 std::istream 的输入流包装器
typedef BasicIStreamWrapper<std::istream> IStreamWrapper;
// 定义基于 std::wistream 的输入流包装器
typedef BasicIStreamWrapper<std::wistream> WIStreamWrapper;

// 弹出之前的诊断状态
#if defined(__clang__) || defined(_MSC_VER)
RAPIDJSON_DIAG_POP
#endif

// 命名空间结束
RAPIDJSON_NAMESPACE_END

// 结束条件编译
#endif // RAPIDJSON_ISTREAMWRAPPER_H_
```