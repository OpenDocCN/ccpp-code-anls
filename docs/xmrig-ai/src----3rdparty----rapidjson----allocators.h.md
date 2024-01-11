# `xmrig\src\3rdparty\rapidjson\allocators.h`

```
#ifndef RAPIDJSON_ALLOCATORS_H_
#define RAPIDJSON_ALLOCATORS_H_

#include "rapidjson.h" // 包含 rapidjson.h 头文件
#include "internal/meta.h" // 包含 internal/meta.h 头文件

#include <memory> // 包含 memory 头文件

#if RAPIDJSON_HAS_CXX11 // 如果定义了 RAPIDJSON_HAS_CXX11
#include <type_traits> // 包含 type_traits 头文件
#endif

RAPIDJSON_NAMESPACE_BEGIN // 开始 rapidjson 命名空间

///////////////////////////////////////////////////////////////////////////////
// Allocator

/*! \class rapidjson::Allocator
    \brief Concept for allocating, resizing and freeing memory block.

    Note that Malloc() and Realloc() are non-static but Free() is static.

    So if an allocator need to support Free(), it needs to put its pointer in
    the header of memory block.

\code
concept Allocator {
    static const bool kNeedFree;    //!< Whether this allocator needs to call Free().

    // Allocate a memory block.
    // \param size of the memory block in bytes.
    // \returns pointer to the memory block.
    void* Malloc(size_t size);

    // Resize a memory block.
    // \param originalPtr The pointer to current memory block. Null pointer is permitted.
    // \param originalSize The current size in bytes. (Design issue: since some allocator may not book-keep this, explicitly pass to it can save memory.)
    // \param newSize the new size in bytes.
    void* Realloc(void* originalPtr, size_t originalSize, size_t newSize);

    // Free a memory block.
    # 定义一个静态函数，用于释放内存块
    # 参数为指向内存块的指针。允许空指针。
    static void Free(void *ptr);
// 定义默认的内存块容量，用户可以自定义为任何2的幂大小
#ifndef RAPIDJSON_ALLOCATOR_DEFAULT_CHUNK_CAPACITY
#define RAPIDJSON_ALLOCATOR_DEFAULT_CHUNK_CAPACITY (64 * 1024)
#endif

///////////////////////////////////////////////////////////////////////////////
// CrtAllocator

// C运行时库分配器，是标准C库内存例程的包装器，实现了分配器概念
class CrtAllocator {
public:
    static const bool kNeedFree = true;
    // 分配指定大小的内存
    void* Malloc(size_t size) {
        if (size) // malloc(0)的行为是由实现定义的
            return RAPIDJSON_MALLOC(size);
        else
            return NULL; // 标准化为返回NULL
    }
    // 重新分配内存
    void* Realloc(void* originalPtr, size_t originalSize, size_t newSize) {
        (void)originalSize;
        if (newSize == 0) {
            RAPIDJSON_FREE(originalPtr);
            return NULL;
        }
        return RAPIDJSON_REALLOC(originalPtr, newSize);
    }
    // 释放内存
    static void Free(void *ptr) RAPIDJSON_NOEXCEPT { RAPIDJSON_FREE(ptr); }

    // 判断是否相等
    bool operator==(const CrtAllocator&) const RAPIDJSON_NOEXCEPT {
        return true;
    }
    // 判断是否不相等
    bool operator!=(const CrtAllocator&) const RAPIDJSON_NOEXCEPT {
        return false;
    }
};

///////////////////////////////////////////////////////////////////////////////
// MemoryPoolAllocator

// 解析器和DOM使用的默认内存分配器，从预分配的内存块中分配内存
// 不释放内存块，Realloc()只分配新内存
// 内存块由BaseAllocator分配，通常是CrtAllocator
// 用户也可以提供一个缓冲区作为第一个内存块
// 如果用户缓冲区已满，则BaseAllocator会分配额外的内存块
    # 这个分配器不会释放用户缓冲区。
    # BaseAllocator是用于分配内存块的分配器类型。默认为CrtAllocator。
    # 实现了分配器概念
template <typename BaseAllocator = CrtAllocator>
class MemoryPoolAllocator {
    //! Chunk header for perpending to each chunk.
    /*! Chunks are stored as a singly linked list.
    */
    struct ChunkHeader {
        size_t capacity;    //!< Capacity of the chunk in bytes (excluding the header itself).
        size_t size;        //!< Current size of allocated memory in bytes.
        ChunkHeader *next;  //!< Next chunk in the linked list.
    };

    struct SharedData {
        ChunkHeader *chunkHead;  //!< Head of the chunk linked-list. Only the head chunk serves allocation.
        BaseAllocator* ownBaseAllocator; //!< base allocator created by this object.
        size_t refcount;
        bool ownBuffer;
    };

    static const size_t SIZEOF_SHARED_DATA = RAPIDJSON_ALIGN(sizeof(SharedData));
    static const size_t SIZEOF_CHUNK_HEADER = RAPIDJSON_ALIGN(sizeof(ChunkHeader));

    static inline ChunkHeader *GetChunkHead(SharedData *shared)
    {
        return reinterpret_cast<ChunkHeader*>(reinterpret_cast<uint8_t*>(shared) + SIZEOF_SHARED_DATA);
    }
    static inline uint8_t *GetChunkBuffer(SharedData *shared)
    {
        return reinterpret_cast<uint8_t*>(shared->chunkHead) + SIZEOF_CHUNK_HEADER;
    }

    static const size_t kDefaultChunkCapacity = RAPIDJSON_ALLOCATOR_DEFAULT_CHUNK_CAPACITY; //!< Default chunk capacity.

public:
    static const bool kNeedFree = false;    //!< Tell users that no need to call Free() with this allocator. (concept Allocator)
    static const bool kRefCounted = true;   //!< Tell users that this allocator is reference counted on copy

    //! Constructor with chunkSize.
    /*! \param chunkSize The size of memory chunk. The default is kDefaultChunkSize.
        \param baseAllocator The allocator for allocating memory chunks.
    */
    explicit
    # 内存池分配器的构造函数，可以指定块大小和基础分配器
    MemoryPoolAllocator(size_t chunkSize = kDefaultChunkCapacity, BaseAllocator* baseAllocator = 0) :
        # 设置块容量
        chunk_capacity_(chunkSize),
        # 设置基础分配器
        baseAllocator_(baseAllocator ? baseAllocator : RAPIDJSON_NEW(BaseAllocator)()),
        # 分配共享数据的内存
        shared_(static_cast<SharedData*>(baseAllocator_ ? baseAllocator_->Malloc(SIZEOF_SHARED_DATA + SIZEOF_CHUNK_HEADER) : 0))
    {
        # 断言基础分配器不为空
        RAPIDJSON_ASSERT(baseAllocator_ != 0);
        # 断言共享数据不为空
        RAPIDJSON_ASSERT(shared_ != 0);
        # 如果有基础分配器，则将 ownBaseAllocator 设置为 0
        if (baseAllocator) {
            shared_->ownBaseAllocator = 0;
        }
        # 否则将 ownBaseAllocator 设置为 baseAllocator_
        else {
            shared_->ownBaseAllocator = baseAllocator_;
        }
        # 设置块头的容量和大小为 0
        shared_->chunkHead = GetChunkHead(shared_);
        shared_->chunkHead->capacity = 0;
        shared_->chunkHead->size = 0;
        shared_->chunkHead->next = 0;
        # 设置 ownBuffer 为 true
        shared_->ownBuffer = true;
        # 引用计数设置为 1
        shared_->refcount = 1;
    }

    # 带有用户提供缓冲区的构造函数
    MemoryPoolAllocator(void *buffer, size_t size, size_t chunkSize = kDefaultChunkCapacity, BaseAllocator* baseAllocator = 0) :
        # 设置块容量
        chunk_capacity_(chunkSize),
        # 设置基础分配器
        baseAllocator_(baseAllocator),
        # 将用户提供的缓冲区对齐后作为共享数据的内存
        shared_(static_cast<SharedData*>(AlignBuffer(buffer, size)))
    {
        // 确保 size 大于等于共享数据和块头的大小
        RAPIDJSON_ASSERT(size >= SIZEOF_SHARED_DATA + SIZEOF_CHUNK_HEADER);
        // 获取共享数据的块头
        shared_->chunkHead = GetChunkHead(shared_);
        // 设置块头的容量为 size 减去共享数据和块头的大小
        shared_->chunkHead->capacity = size - SIZEOF_SHARED_DATA - SIZEOF_CHUNK_HEADER;
        // 设置块头的大小为 0
        shared_->chunkHead->size = 0;
        // 设置块头的下一个块为 0
        shared_->chunkHead->next = 0;
        // 设置自己的基础分配器为 0
        shared_->ownBaseAllocator = 0;
        // 设置自己的缓冲区为 false
        shared_->ownBuffer = false;
        // 设置引用计数为 1
        shared_->refcount = 1;
    }

    // 复制构造函数
    MemoryPoolAllocator(const MemoryPoolAllocator& rhs) RAPIDJSON_NOEXCEPT :
        // 初始化块容量为 rhs 的块容量
        chunk_capacity_(rhs.chunk_capacity_),
        // 初始化基础分配器为 rhs 的基础分配器
        baseAllocator_(rhs.baseAllocator_),
        // 初始化共享数据为 rhs 的共享数据
        shared_(rhs.shared_)
    {
        // 确保共享数据的引用计数大于 0
        RAPIDJSON_NOEXCEPT_ASSERT(shared_->refcount > 0);
        // 增加共享数据的引用计数
        ++shared_->refcount;
    }
    // 赋值运算符重载
    MemoryPoolAllocator& operator=(const MemoryPoolAllocator& rhs) RAPIDJSON_NOEXCEPT
    {
        // 确保 rhs 的共享数据的引用计数大于 0
        RAPIDJSON_NOEXCEPT_ASSERT(rhs.shared_->refcount > 0);
        // 增加 rhs 的共享数据的引用计数
        ++rhs.shared_->refcount;
        // 调用析构函数
        this->~MemoryPoolAllocator();
        // 将基础分配器设置为 rhs 的基础分配器
        baseAllocator_ = rhs.baseAllocator_;
        // 将块容量设置为 rhs 的块容量
        chunk_capacity_ = rhs.chunk_capacity_;
        // 将共享数据设置为 rhs 的共享数据
        shared_ = rhs.shared_;
        // 返回自身
        return *this;
    }
#if RAPIDJSON_HAS_CXX11_RVALUE_REFS
    // 如果支持 C++11 的右值引用
    MemoryPoolAllocator(MemoryPoolAllocator&& rhs) RAPIDJSON_NOEXCEPT :
        // 移动构造函数，初始化内存块容量、基础分配器和共享指针
        chunk_capacity_(rhs.chunk_capacity_),
        baseAllocator_(rhs.baseAllocator_),
        shared_(rhs.shared_)
    {
        // 断言确保共享指针的引用计数大于 0
        RAPIDJSON_NOEXCEPT_ASSERT(rhs.shared_->refcount > 0);
        // 将原共享指针置为 0
        rhs.shared_ = 0;
    }
    // 移动赋值运算符
    MemoryPoolAllocator& operator=(MemoryPoolAllocator&& rhs) RAPIDJSON_NOEXCEPT
    {
        // 断言确保共享指针的引用计数大于 0
        RAPIDJSON_NOEXCEPT_ASSERT(rhs.shared_->refcount > 0);
        // 调用析构函数
        this->~MemoryPoolAllocator();
        // 赋值基础分配器、内存块容量和共享指针
        baseAllocator_ = rhs.baseAllocator_;
        chunk_capacity_ = rhs.chunk_capacity_;
        shared_ = rhs.shared_;
        // 将原共享指针置为 0
        rhs.shared_ = 0;
        return *this;
    }
#endif

    //! 析构函数
    /*! 释放所有内存块，不包括用户提供的缓冲区
    */
    ~MemoryPoolAllocator() RAPIDJSON_NOEXCEPT {
        if (!shared_) {
            // 如果已移动，则不执行任何操作
            return;
        }
        if (shared_->refcount > 1) {
            // 如果引用计数大于 1，则递减引用计数并返回
            --shared_->refcount;
            return;
        }
        // 清空内存
        Clear();
        BaseAllocator *a = shared_->ownBaseAllocator;
        if (shared_->ownBuffer) {
            baseAllocator_->Free(shared_);
        }
        RAPIDJSON_DELETE(a);
    }

    //! 清空所有内存块，不包括第一个/用户提供的内存块
    void Clear() RAPIDJSON_NOEXCEPT {
        // 断言确保共享指针的引用计数大于 0
        RAPIDJSON_NOEXCEPT_ASSERT(shared_->refcount > 0);
        for (;;) {
            ChunkHeader* c = shared_->chunkHead;
            if (!c->next) {
                break;
            }
            shared_->chunkHead = c->next;
            baseAllocator_->Free(c);
        }
        shared_->chunkHead->size = 0;
    }

    //! 计算已分配内存块的总容量
    /*! \return 总容量（字节）
    */
    # 返回分配器的总容量
    size_t Capacity() const RAPIDJSON_NOEXCEPT {
        # 断言共享指针的引用计数大于0
        RAPIDJSON_NOEXCEPT_ASSERT(shared_->refcount > 0);
        # 初始化容量为0
        size_t capacity = 0;
        # 遍历所有内存块，累加容量
        for (ChunkHeader* c = shared_->chunkHead; c != 0; c = c->next)
            capacity += c->capacity;
        # 返回总容量
        return capacity;
    }

    # 计算已分配的内存块大小
    /*! \return 已使用的总字节数.
    */
    size_t Size() const RAPIDJSON_NOEXCEPT {
        # 断言共享指针的引用计数大于0
        RAPIDJSON_NOEXCEPT_ASSERT(shared_->refcount > 0);
        # 初始化大小为0
        size_t size = 0;
        # 遍历所有内存块，累加大小
        for (ChunkHeader* c = shared_->chunkHead; c != 0; c = c->next)
            size += c->size;
        # 返回总大小
        return size;
    }

    # 判断分配器是否是共享的
    /*! \return true 或 false.
    */
    bool Shared() const RAPIDJSON_NOEXCEPT {
        # 断言共享指针的引用计数大于0
        RAPIDJSON_NOEXCEPT_ASSERT(shared_->refcount > 0);
        # 返回引用计数是否大于1
        return shared_->refcount > 1;
    }

    # 分配内存块 (分配器概念)
    void* Malloc(size_t size) {
        # 断言共享指针的引用计数大于0
        RAPIDJSON_NOEXCEPT_ASSERT(shared_->refcount > 0);
        # 如果size为0，返回空指针
        if (!size)
            return NULL;

        # 对齐size
        size = RAPIDJSON_ALIGN(size);
        # 如果当前内存块的大小加上size大于当前内存块的容量
        if (RAPIDJSON_UNLIKELY(shared_->chunkHead->size + size > shared_->chunkHead->capacity))
            # 如果无法添加新的内存块，返回空指针
            if (!AddChunk(chunk_capacity_ > size ? chunk_capacity_ : size))
                return NULL;

        # 获取内存块的缓冲区
        void *buffer = GetChunkBuffer(shared_) + shared_->chunkHead->size;
        # 更新内存块的大小
        shared_->chunkHead->size += size;
        # 返回缓冲区
        return buffer;
    }

    # 调整内存块大小 (分配器概念)
    // 重新分配内存，用于调整原始指针指向的内存块的大小
    void* Realloc(void* originalPtr, size_t originalSize, size_t newSize) {
        // 如果原始指针为空，则分配新的内存
        if (originalPtr == 0)
            return Malloc(newSize);

        // 断言共享指针的引用计数大于0
        RAPIDJSON_NOEXCEPT_ASSERT(shared_->refcount > 0);
        // 如果新大小为0，则返回空指针
        if (newSize == 0)
            return NULL;

        // 对原始大小和新大小进行对齐
        originalSize = RAPIDJSON_ALIGN(originalSize);
        newSize = RAPIDJSON_ALIGN(newSize);

        // 如果新大小小于等于原始大小，则不进行缩小
        if (originalSize >= newSize)
            return originalPtr;

        // 如果原始指针指向的内存块是最后一个分配的，并且有足够的空间，则简单地扩展它
        if (originalPtr == GetChunkBuffer(shared_) + shared_->chunkHead->size - originalSize) {
            size_t increment = static_cast<size_t>(newSize - originalSize);
            if (shared_->chunkHead->size + increment <= shared_->chunkHead->capacity) {
                shared_->chunkHead->size += increment;
                return originalPtr;
            }
        }

        // 重新分配过程：分配并复制内存，不释放原始缓冲区
        if (void* newBuffer = Malloc(newSize)) {
            if (originalSize)
                std::memcpy(newBuffer, originalPtr, originalSize);
            return newBuffer;
        }
        else
            return NULL;
    }

    //! 释放内存块（分配器概念）
    static void Free(void *ptr) RAPIDJSON_NOEXCEPT { (void)ptr; } // 什么也不做

    //! 与另一个MemoryPoolAllocator进行比较（相等性）
    bool operator==(const MemoryPoolAllocator& rhs) const RAPIDJSON_NOEXCEPT {
        RAPIDJSON_NOEXCEPT_ASSERT(shared_->refcount > 0);
        RAPIDJSON_NOEXCEPT_ASSERT(rhs.shared_->refcount > 0);
        return shared_ == rhs.shared_;
    }
    //! 与另一个MemoryPoolAllocator进行比较（不相等）
    bool operator!=(const MemoryPoolAllocator& rhs) const RAPIDJSON_NOEXCEPT {
        return !operator==(rhs);
    }
private:
    //! 创建一个新的数据块。
    /*! \param capacity 数据块的容量（以字节为单位）。
        \return 如果成功则返回true。
    */
    bool AddChunk(size_t capacity) {
        // 如果baseAllocator_为空，则创建一个新的BaseAllocator对象
        if (!baseAllocator_)
            shared_->ownBaseAllocator = baseAllocator_ = RAPIDJSON_NEW(BaseAllocator)();
        // 分配内存给新的数据块
        if (ChunkHeader* chunk = static_cast<ChunkHeader*>(baseAllocator_->Malloc(SIZEOF_CHUNK_HEADER + capacity))) {
            chunk->capacity = capacity;
            chunk->size = 0;
            chunk->next = shared_->chunkHead;
            shared_->chunkHead = chunk;
            return true;
        }
        else
            return false;
    }

    // 对齐缓冲区的内存地址
    static inline void* AlignBuffer(void* buf, size_t &size)
    {
        RAPIDJSON_NOEXCEPT_ASSERT(buf != 0);
        const uintptr_t mask = sizeof(void*) - 1;
        const uintptr_t ubuf = reinterpret_cast<uintptr_t>(buf);
        if (RAPIDJSON_UNLIKELY(ubuf & mask)) {
            const uintptr_t abuf = (ubuf + mask) & ~mask;
            RAPIDJSON_ASSERT(size >= abuf - ubuf);
            buf = reinterpret_cast<void*>(abuf);
            size -= abuf - ubuf;
        }
        return buf;
    }

    size_t chunk_capacity_;     //!< 分配内存块的最小容量。
    BaseAllocator* baseAllocator_;  //!< 用于分配内存块的基础分配器。
    SharedData *shared_;        //!< 分配器的共享数据。
};

namespace internal {
    // 检查类型是否具有引用计数
    template<typename, typename = void>
    struct IsRefCounted :
        public FalseType
    { };
    template<typename T>
    struct IsRefCounted<T, typename internal::EnableIfCond<T::kRefCounted>::Type> :
        public TrueType
    { };
}

// 重新分配内存
template<typename T, typename A>
inline T* Realloc(A& a, T* old_p, size_t old_n, size_t new_n)
{
    RAPIDJSON_NOEXCEPT_ASSERT(old_n <= SIZE_MAX / sizeof(T) && new_n <= SIZE_MAX / sizeof(T));
    return static_cast<T*>(a.Realloc(old_p, old_n * sizeof(T), new_n * sizeof(T)));
}

template<typename T, typename A>
// 分配内存并返回指向类型 T 的指针
inline T *Malloc(A& a, size_t n = 1)
{
    return Realloc<T, A>(a, NULL, 0, n);
}

// 释放由指针 p 指向的内存
template<typename T, typename A>
inline void Free(A& a, T *p, size_t n = 1)
{
    static_cast<void>(Realloc<T, A>(a, p, n, 0));
}

#ifdef __GNUC__
RAPIDJSON_DIAG_PUSH
RAPIDJSON_DIAG_OFF(effc++) // std::allocator can safely be inherited
#endif

// 定义 StdAllocator 类模板，继承自 std::allocator<T>
template <typename T, typename BaseAllocator = CrtAllocator>
class StdAllocator :
    public std::allocator<T>
{
    typedef std::allocator<T> allocator_type;
#if RAPIDJSON_HAS_CXX11
    typedef std::allocator_traits<allocator_type> traits_type;
#else
    typedef allocator_type traits_type;
#endif

public:
    typedef BaseAllocator BaseAllocatorType;

    // 默认构造函数
    StdAllocator() RAPIDJSON_NOEXCEPT :
        allocator_type(),
        baseAllocator_()
    { }

    // 拷贝构造函数
    StdAllocator(const StdAllocator& rhs) RAPIDJSON_NOEXCEPT :
        allocator_type(rhs),
        baseAllocator_(rhs.baseAllocator_)
    { }

    // 模板拷贝构造函数
    template<typename U>
    StdAllocator(const StdAllocator<U, BaseAllocator>& rhs) RAPIDJSON_NOEXCEPT :
        allocator_type(rhs),
        baseAllocator_(rhs.baseAllocator_)
    { }

#if RAPIDJSON_HAS_CXX11_RVALUE_REFS
    // 移动构造函数
    StdAllocator(StdAllocator&& rhs) RAPIDJSON_NOEXCEPT :
        allocator_type(std::move(rhs)),
        baseAllocator_(std::move(rhs.baseAllocator_))
    { }
#endif
#if RAPIDJSON_HAS_CXX11
    using propagate_on_container_move_assignment = std::true_type;
    using propagate_on_container_swap = std::true_type;
#endif

    /* implicit */
    // 隐式构造函数
    StdAllocator(const BaseAllocator& allocator) RAPIDJSON_NOEXCEPT :
        allocator_type(),
        baseAllocator_(allocator)
    { }

    // 析构函数
    ~StdAllocator() RAPIDJSON_NOEXCEPT
    { }

    // 重新绑定模板参数
    template<typename U>
    struct rebind {
        typedef StdAllocator<U, BaseAllocator> other;
    };

    // 定义类型别名
    typedef typename traits_type::size_type         size_type;
    typedef typename traits_type::difference_type   difference_type;

    typedef typename traits_type::value_type        value_type;
    # 使用模板参数中的 traits_type 定义指针类型 pointer
    typedef typename traits_type::pointer           pointer;
    # 使用模板参数中的 traits_type 定义常量指针类型 const_pointer
    typedef typename traits_type::const_pointer     const_pointer;
#if RAPIDJSON_HAS_CXX11
    // 定义引用类型为 value_type 的别名
    typedef typename std::add_lvalue_reference<value_type>::type &reference;
    // 定义常量引用类型为 value_type 的别名
    typedef typename std::add_lvalue_reference<typename std::add_const<value_type>::type>::type &const_reference;

    // 返回引用的地址
    pointer address(reference r) const RAPIDJSON_NOEXCEPT
    {
        return std::addressof(r);
    }
    // 返回常量引用的地址
    const_pointer address(const_reference r) const RAPIDJSON_NOEXCEPT
    {
        return std::addressof(r);
    }

    // 返回分配器的最大大小
    size_type max_size() const RAPIDJSON_NOEXCEPT
    {
        return traits_type::max_size(*this);
    }

    // 构造对象
    template <typename ...Args>
    void construct(pointer p, Args&&... args)
    {
        traits_type::construct(*this, p, std::forward<Args>(args)...);
    }
    // 销毁对象
    void destroy(pointer p)
    {
        traits_type::destroy(*this, p);
    }

#else // !RAPIDJSON_HAS_CXX11
    // 定义引用类型为 allocator_type::reference 的别名
    typedef typename allocator_type::reference       reference;
    // 定义常量引用类型为 allocator_type::const_reference 的别名
    typedef typename allocator_type::const_reference const_reference;

    // 返回引用的地址
    pointer address(reference r) const RAPIDJSON_NOEXCEPT
    {
        return allocator_type::address(r);
    }
    // 返回常量引用的地址
    const_pointer address(const_reference r) const RAPIDJSON_NOEXCEPT
    {
        return allocator_type::address(r);
    }

    // 返回分配器的最大大小
    size_type max_size() const RAPIDJSON_NOEXCEPT
    {
        return allocator_type::max_size();
    }

    // 构造对象
    void construct(pointer p, const_reference r)
    {
        allocator_type::construct(p, r);
    }
    // 销毁对象
    void destroy(pointer p)
    {
        allocator_type::destroy(p);
    }

#endif // !RAPIDJSON_HAS_CXX11

    // 分配内存
    template <typename U>
    U* allocate(size_type n = 1, const void* = 0)
    {
        return RAPIDJSON_NAMESPACE::Malloc<U>(baseAllocator_, n);
    }
    // 释放内存
    template <typename U>
    void deallocate(U* p, size_type n = 1)
    {
        RAPIDJSON_NAMESPACE::Free<U>(baseAllocator_, p, n);
    }

    // 分配内存
    pointer allocate(size_type n = 1, const void* = 0)
    {
        return allocate<value_type>(n);
    }
    // 释放内存
    void deallocate(pointer p, size_type n = 1)
    # 调用 deallocate 函数释放指针 p 指向的内存，释放 n 个 value_type 类型的对象
    deallocate<value_type>(p, n);
#if RAPIDJSON_HAS_CXX11
    // 如果支持 C++11，则定义 is_always_equal 类型为 std::is_empty<BaseAllocator> 的结果
    using is_always_equal = std::is_empty<BaseAllocator>;
#endif

    // 比较当前 StdAllocator 对象和另一个 StdAllocator 对象是否相等
    template<typename U>
    bool operator==(const StdAllocator<U, BaseAllocator>& rhs) const RAPIDJSON_NOEXCEPT
    {
        return baseAllocator_ == rhs.baseAllocator_;
    }
    // 比较当前 StdAllocator 对象和另一个 StdAllocator 对象是否不相等
    template<typename U>
    bool operator!=(const StdAllocator<U, BaseAllocator>& rhs) const RAPIDJSON_NOEXCEPT
    {
        return !operator==(rhs);
    }

    //! rapidjson Allocator concept
    // 定义常量 kNeedFree 为 BaseAllocator::kNeedFree
    static const bool kNeedFree = BaseAllocator::kNeedFree;
    // 定义常量 kRefCounted 为 internal::IsRefCounted<BaseAllocator>::Value
    static const bool kRefCounted = internal::IsRefCounted<BaseAllocator>::Value;
    // 分配内存
    void* Malloc(size_t size)
    {
        return baseAllocator_.Malloc(size);
    }
    // 重新分配内存
    void* Realloc(void* originalPtr, size_t originalSize, size_t newSize)
    {
        return baseAllocator_.Realloc(originalPtr, originalSize, newSize);
    }
    // 释放内存
    static void Free(void *ptr) RAPIDJSON_NOEXCEPT
    {
        BaseAllocator::Free(ptr);
    }

private:
    // 友元类模板，用于访问 StdAllocator<!T>.*
    template <typename, typename>
    friend class StdAllocator; // access to StdAllocator<!T>.*

    // 基础分配器对象
    BaseAllocator baseAllocator_;
};

#if !RAPIDJSON_HAS_CXX17 // std::allocator<void> deprecated in C++17
// 针对 void 类型的特化模板
template <typename BaseAllocator>
class StdAllocator<void, BaseAllocator> :
    public std::allocator<void>
{
    typedef std::allocator<void> allocator_type;

public:
    typedef BaseAllocator BaseAllocatorType;

    // 默认构造函数
    StdAllocator() RAPIDJSON_NOEXCEPT :
        allocator_type(),
        baseAllocator_()
    { }

    // 拷贝构造函数
    StdAllocator(const StdAllocator& rhs) RAPIDJSON_NOEXCEPT :
        allocator_type(rhs),
        baseAllocator_(rhs.baseAllocator_)
    { }

    // 模板拷贝构造函数
    template<typename U>
    StdAllocator(const StdAllocator<U, BaseAllocator>& rhs) RAPIDJSON_NOEXCEPT :
        allocator_type(rhs),
        baseAllocator_(rhs.baseAllocator_)
    { }

    /* implicit */
    // 隐式构造函数
    StdAllocator(const BaseAllocator& baseAllocator) RAPIDJSON_NOEXCEPT :
        allocator_type(),
        baseAllocator_(baseAllocator)
    { }
    # 析构函数，使用默认参数 RAPIDJSON_NOEXCEPT
    ~StdAllocator() RAPIDJSON_NOEXCEPT
    { }

    # 重新绑定模板，将 U 类型重新绑定为 StdAllocator<U, BaseAllocator> 类型
    template<typename U>
    struct rebind {
        typedef StdAllocator<U, BaseAllocator> other;
    };

    # 定义 value_type 类型为 allocator_type 的 value_type
    typedef typename allocator_type::value_type value_type;
private:
    template <typename, typename>
    friend class StdAllocator; // access to StdAllocator<!T>.*

    BaseAllocator baseAllocator_;
};
#endif

#ifdef __GNUC__
RAPIDJSON_DIAG_POP
#endif

RAPIDJSON_NAMESPACE_END

#endif // RAPIDJSON_ENCODINGS_H_
```