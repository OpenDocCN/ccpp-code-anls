# `xmrig\src\3rdparty\rapidjson\memorybuffer.h`

```
#ifndef RAPIDJSON_MEMORYBUFFER_H_
#define RAPIDJSON_MEMORYBUFFER_H_

#include "stream.h"
#include "internal/stack.h"

RAPIDJSON_NAMESPACE_BEGIN

//! Represents an in-memory output byte stream.
/*!
    This class is mainly for being wrapped by EncodedOutputStream or AutoUTFOutputStream.

    It is similar to FileWriteBuffer but the destination is an in-memory buffer instead of a file.

    Differences between MemoryBuffer and StringBuffer:
    1. StringBuffer has Encoding but MemoryBuffer is only a byte buffer.
    2. StringBuffer::GetString() returns a null-terminated string. MemoryBuffer::GetBuffer() returns a buffer without terminator.

    \tparam Allocator type for allocating memory buffer.
    \note implements Stream concept
*/
template <typename Allocator = CrtAllocator>
struct GenericMemoryBuffer {
    typedef char Ch; // byte

    GenericMemoryBuffer(Allocator* allocator = 0, size_t capacity = kDefaultCapacity) : stack_(allocator, capacity) {}

    // 将字符写入内存缓冲区
    void Put(Ch c) { *stack_.template Push<Ch>() = c; }
    // 刷新缓冲区，内存缓冲区无需刷新
    void Flush() {}

    // 清空内存缓冲区
    void Clear() { stack_.Clear(); }
    // 将内存缓冲区的容量调整为实际大小
    void ShrinkToFit() { stack_.ShrinkToFit(); }
    // 在内存缓冲区中分配指定数量的字节，并返回指向分配内存的指针
    Ch* Push(size_t count) { return stack_.template Push<Ch>(count); }
    // 从内存缓冲区中弹出指定数量的字节
    void Pop(size_t count) { stack_.template Pop<Ch>(count); }
    # 返回栈底指针，即缓冲区的起始地址
    const Ch* GetBuffer() const {
        return stack_.template Bottom<Ch>();
    }

    # 返回栈的大小
    size_t GetSize() const { return stack_.GetSize(); }

    # 默认容量大小为256
    static const size_t kDefaultCapacity = 256;
    
    # 可变的内部栈，使用模板参数 Allocator
    mutable internal::Stack<Allocator> stack_;
};

typedef GenericMemoryBuffer<> MemoryBuffer;

//! Implement specialized version of PutN() with memset() for better performance.
// 为了提高性能，使用memset()实现PutN()的特殊版本
template<>
inline void PutN(MemoryBuffer& memoryBuffer, char c, size_t n) {
    // 使用memset()将字符c填充到内存缓冲区中
    std::memset(memoryBuffer.stack_.Push<char>(n), c, n * sizeof(c));
}

RAPIDJSON_NAMESPACE_END

#endif // RAPIDJSON_MEMORYBUFFER_H_
```