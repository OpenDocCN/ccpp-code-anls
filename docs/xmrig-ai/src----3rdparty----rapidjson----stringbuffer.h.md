# `xmrig\src\3rdparty\rapidjson\stringbuffer.h`

```cpp
// 定义了一个名为 GenericStringBuffer 的模板类，表示一个内存中的输出流
/*!
    \tparam Encoding 流的编码类型
    \tparam Allocator 用于分配内存缓冲区的类型
    \note 实现了流的概念
*/
template <typename Encoding, typename Allocator = CrtAllocator>
class GenericStringBuffer {
public:
    typedef typename Encoding::Ch Ch; // 使用流的编码类型定义 Ch 类型

    // 构造函数，初始化内存分配器和容量
    GenericStringBuffer(Allocator* allocator = 0, size_t capacity = kDefaultCapacity) : stack_(allocator, capacity) {}

#if RAPIDJSON_HAS_CXX11_RVALUE_REFS
    // 移动构造函数，使用 std::move 移动内存分配器
    GenericStringBuffer(GenericStringBuffer&& rhs) : stack_(std::move(rhs.stack_)) {}
    // 移动赋值运算符，使用 std::move 移动内存分配器
    GenericStringBuffer& operator=(GenericStringBuffer&& rhs) {
        if (&rhs != this)
            stack_ = std::move(rhs.stack_);
        return *this;
    }
#endif

    // 向缓冲区中添加一个字符
    void Put(Ch c) { *stack_.template Push<Ch>() = c; }
    // 向缓冲区中添加一个字符，不进行边界检查
    void PutUnsafe(Ch c) { *stack_.template PushUnsafe<Ch>() = c; }
    // 刷新缓冲区，无操作
    void Flush() {}

    // 清空缓冲区
    void Clear() { stack_.Clear(); }
    // 压缩字符串，使其适应当前大小
    void ShrinkToFit() {
        // 压入和弹出一个空终止符。这是安全的。
        *stack_.template Push<Ch>() = '\0';
        // 压缩字符串，使其适应当前大小
        stack_.ShrinkToFit();
        // 弹出一个字符
        stack_.template Pop<Ch>(1);
    }

    // 为字符串预留空间
    void Reserve(size_t count) { stack_.template Reserve<Ch>(count); }
    
    // 压入指定数量的字符
    Ch* Push(size_t count) { return stack_.template Push<Ch>(count); }
    
    // 不安全地压入指定数量的字符
    Ch* PushUnsafe(size_t count) { return stack_.template PushUnsafe<Ch>(count); }
    
    // 弹出指定数量的字符
    void Pop(size_t count) { stack_.template Pop<Ch>(count); }

    // 获取字符串的指针
    const Ch* GetString() const {
        // 压入和弹出一个空终止符。这是安全的。
        *stack_.template Push<Ch>() = '\0';
        // 弹出一个字符
        stack_.template Pop<Ch>(1);

        return stack_.template Bottom<Ch>();
    }

    //! 获取字符串缓冲区中字符串的字节大小
    size_t GetSize() const { return stack_.GetSize(); }

    //! 获取字符串缓冲区中字符串的字符长度
    size_t GetLength() const { return stack_.GetSize() / sizeof(Ch); }

    // 默认容量
    static const size_t kDefaultCapacity = 256;
    // 可变的内部堆栈
    mutable internal::Stack<Allocator> stack_;
private:
    // 禁止复制构造函数和赋值运算符。
    GenericStringBuffer(const GenericStringBuffer&);
    GenericStringBuffer& operator=(const GenericStringBuffer&);
};

//! 使用UTF8编码的字符串缓冲区
typedef GenericStringBuffer<UTF8<> > StringBuffer;

template<typename Encoding, typename Allocator>
inline void PutReserve(GenericStringBuffer<Encoding, Allocator>& stream, size_t count) {
    // 为字符串缓冲区预留足够的空间
    stream.Reserve(count);
}

template<typename Encoding, typename Allocator>
inline void PutUnsafe(GenericStringBuffer<Encoding, Allocator>& stream, typename Encoding::Ch c) {
    // 将字符添加到字符串缓冲区，不进行安全检查
    stream.PutUnsafe(c);
}

//! 使用memset()实现PutN()的特殊版本，以获得更好的性能。
template<>
inline void PutN(GenericStringBuffer<UTF8<> >& stream, char c, size_t n) {
    // 使用memset()将字符c复制n次到字符串缓冲区
    std::memset(stream.stack_.Push<char>(n), c, n * sizeof(c));
}

RAPIDJSON_NAMESPACE_END

#if defined(__clang__)
RAPIDJSON_DIAG_POP
#endif

#endif // RAPIDJSON_STRINGBUFFER_H_
```