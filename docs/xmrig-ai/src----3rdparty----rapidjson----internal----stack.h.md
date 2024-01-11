# `xmrig\src\3rdparty\rapidjson\internal\stack.h`

```
// 定义了一个名为 Stack 的模板类，用于存储不同类型的数据
// 模板参数 Allocator 用于分配栈内存
template <typename Allocator>
class Stack {
public:
    // 优化说明：在构造函数中不要为 stack_ 分配内存
    // 当第一次调用 Push() -> Expand() -> Resize() 时才进行延迟分配内存
    Stack(Allocator* allocator, size_t stackCapacity) : allocator_(allocator), ownAllocator_(0), stack_(0), stackTop_(0), stackEnd_(0), initialCapacity_(stackCapacity) {
    }

    // 使用 C++11 的右值引用实现移动构造函数
    // 用于将 rhs 的资源移动到当前对象
    Stack(Stack&& rhs)
        : allocator_(rhs.allocator_),
          ownAllocator_(rhs.ownAllocator_),
          stack_(rhs.stack_),
          stackTop_(rhs.stackTop_),
          stackEnd_(rhs.stackEnd_),
          initialCapacity_(rhs.initialCapacity_)
    {
        # 将 rhs 对象的 allocator_ 属性设置为 0
        rhs.allocator_ = 0;
        # 将 rhs 对象的 ownAllocator_ 属性设置为 0
        rhs.ownAllocator_ = 0;
        # 将 rhs 对象的 stack_ 属性设置为 0
        rhs.stack_ = 0;
        # 将 rhs 对象的 stackTop_ 属性设置为 0
        rhs.stackTop_ = 0;
        # 将 rhs 对象的 stackEnd_ 属性设置为 0
        rhs.stackEnd_ = 0;
        # 将 rhs 对象的 initialCapacity_ 属性设置为 0
        rhs.initialCapacity_ = 0;
    }
#endif

    // 析构函数，销毁栈
    ~Stack() {
        Destroy();
    }

#if RAPIDJSON_HAS_CXX11_RVALUE_REFS
    // 移动赋值运算符
    Stack& operator=(Stack&& rhs) {
        if (&rhs != this)
        {
            // 销毁当前对象
            Destroy();

            // 移动赋值
            allocator_ = rhs.allocator_;
            ownAllocator_ = rhs.ownAllocator_;
            stack_ = rhs.stack_;
            stackTop_ = rhs.stackTop_;
            stackEnd_ = rhs.stackEnd_;
            initialCapacity_ = rhs.initialCapacity_;

            // 将原对象置空
            rhs.allocator_ = 0;
            rhs.ownAllocator_ = 0;
            rhs.stack_ = 0;
            rhs.stackTop_ = 0;
            rhs.stackEnd_ = 0;
            rhs.initialCapacity_ = 0;
        }
        return *this;
    }
#endif

    // 交换两个栈的内容
    void Swap(Stack& rhs) RAPIDJSON_NOEXCEPT {
        internal::Swap(allocator_, rhs.allocator_);
        internal::Swap(ownAllocator_, rhs.ownAllocator_);
        internal::Swap(stack_, rhs.stack_);
        internal::Swap(stackTop_, rhs.stackTop_);
        internal::Swap(stackEnd_, rhs.stackEnd_);
        internal::Swap(initialCapacity_, rhs.initialCapacity_);
    }

    // 清空栈
    void Clear() { stackTop_ = stack_; }

    // 将栈的容量调整为实际使用的大小
    void ShrinkToFit() {
        if (Empty()) {
            // 如果栈为空，完全释放内存
            Allocator::Free(stack_); // NOLINT (+clang-analyzer-unix.Malloc)
            stack_ = 0;
            stackTop_ = 0;
            stackEnd_ = 0;
        }
        else
            Resize(GetSize());
    }

    // 为栈预留空间
    // 优化说明：尽量减小此函数的大小以进行强制内联。
    // 扩展运行非常不频繁，因此将其移动到另一个（可能非内联）函数中。
    template<typename T>
    RAPIDJSON_FORCEINLINE void Reserve(size_t count = 1) {
         // 如果需要，扩展栈的大小
        if (RAPIDJSON_UNLIKELY(static_cast<std::ptrdiff_t>(sizeof(T) * count) > (stackEnd_ - stackTop_)))
            Expand<T>(count);
    }

    template<typename T>
    # 在栈顶推入指定数量的元素，并确保有足够的空间
    RAPIDJSON_FORCEINLINE T* Push(size_t count = 1) {
        Reserve<T>(count);
        return PushUnsafe<T>(count);
    }

    # 在栈顶推入指定数量的元素，不进行空间检查
    template<typename T>
    RAPIDJSON_FORCEINLINE T* PushUnsafe(size_t count = 1) {
        RAPIDJSON_ASSERT(stackTop_);
        RAPIDJSON_ASSERT(static_cast<std::ptrdiff_t>(sizeof(T) * count) <= (stackEnd_ - stackTop_));
        T* ret = reinterpret_cast<T*>(stackTop_);
        stackTop_ += sizeof(T) * count;
        return ret;
    }

    # 从栈顶弹出指定数量的元素
    template<typename T>
    T* Pop(size_t count) {
        RAPIDJSON_ASSERT(GetSize() >= count * sizeof(T));
        stackTop_ -= count * sizeof(T);
        return reinterpret_cast<T*>(stackTop_);
    }

    # 返回栈顶元素的指针
    template<typename T>
    T* Top() {
        RAPIDJSON_ASSERT(GetSize() >= sizeof(T));
        return reinterpret_cast<T*>(stackTop_ - sizeof(T));
    }

    # 返回栈顶元素的指针（const 重载）
    template<typename T>
    const T* Top() const {
        RAPIDJSON_ASSERT(GetSize() >= sizeof(T));
        return reinterpret_cast<T*>(stackTop_ - sizeof(T));
    }

    # 返回栈顶之后的元素的指针
    template<typename T>
    T* End() { return reinterpret_cast<T*>(stackTop_); }

    # 返回栈顶之后的元素的指针（const 重载）
    template<typename T>
    const T* End() const { return reinterpret_cast<T*>(stackTop_); }

    # 返回栈底的元素的指针
    template<typename T>
    T* Bottom() { return reinterpret_cast<T*>(stack_); }

    # 返回栈底的元素的指针（const 重载）
    template<typename T>
    const T* Bottom() const { return reinterpret_cast<T*>(stack_); }

    # 检查是否有分配器
    bool HasAllocator() const {
        return allocator_ != 0;
    }

    # 获取分配器的引用
    Allocator& GetAllocator() {
        RAPIDJSON_ASSERT(allocator_);
        return *allocator_;
    }

    # 检查栈是否为空
    bool Empty() const { return stackTop_ == stack_; }

    # 获取栈中元素的大小
    size_t GetSize() const { return static_cast<size_t>(stackTop_ - stack_); }

    # 获取栈的容量
    size_t GetCapacity() const { return static_cast<size_t>(stackEnd_ - stack_); }
private:
    template<typename T>
    void Expand(size_t count) {
        // 只有在当前栈存在时才扩展容量。否则，只需创建一个具有初始容量的栈。
        size_t newCapacity;
        if (stack_ == 0) {
            if (!allocator_)
                ownAllocator_ = allocator_ = RAPIDJSON_NEW(Allocator)();  // 如果分配器不存在，则创建一个新的分配器
            newCapacity = initialCapacity_;  // 初始容量
        } else {
            newCapacity = GetCapacity();  // 获取当前容量
            newCapacity += (newCapacity + 1) / 2;  // 扩展容量为当前容量的1.5倍
        }
        size_t newSize = GetSize() + sizeof(T) * count;  // 计算新的大小
        if (newCapacity < newSize)
            newCapacity = newSize;  // 如果新容量小于新大小，则将新容量设置为新大小

        Resize(newCapacity);  // 调整栈的容量
    }

    void Resize(size_t newCapacity) {
        const size_t size = GetSize();  // 备份当前大小
        stack_ = static_cast<char*>(allocator_->Realloc(stack_, GetCapacity(), newCapacity));  // 重新分配栈的内存空间
        stackTop_ = stack_ + size;  // 设置栈顶指针
        stackEnd_ = stack_ + newCapacity;  // 设置栈的结束指针
    }

    void Destroy() {
        Allocator::Free(stack_);  // 释放栈的内存空间
        RAPIDJSON_DELETE(ownAllocator_); // 只有在栈拥有分配器时才删除
    }

    // 禁止复制构造函数和赋值运算符。
    Stack(const Stack&);  // 禁止复制构造函数
    Stack& operator=(const Stack&);  // 禁止赋值运算符

    Allocator* allocator_;  // 分配器指针
    Allocator* ownAllocator_;  // 拥有的分配器指针
    char *stack_;  // 栈指针
    char *stackTop_;  // 栈顶指针
    char *stackEnd_;  // 栈结束指针
    size_t initialCapacity_;  // 初始容量
};

} // namespace internal
RAPIDJSON_NAMESPACE_END

#if defined(__clang__)
RAPIDJSON_DIAG_POP
#endif

#endif // RAPIDJSON_STACK_H_
```