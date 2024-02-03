# `xmrig\src\3rdparty\rapidjson\uri.h`

```cpp
#ifndef RAPIDJSON_URI_H_
#define RAPIDJSON_URI_H_

#include "internal/strfunc.h"

#if defined(__clang__)
RAPIDJSON_DIAG_PUSH
RAPIDJSON_DIAG_OFF(c++98-compat)
#elif defined(_MSC_VER)
RAPIDJSON_DIAG_OFF(4512) // assignment operator could not be generated
#endif

RAPIDJSON_NAMESPACE_BEGIN

///////////////////////////////////////////////////////////////////////////////
// GenericUri

template <typename ValueType, typename Allocator=CrtAllocator>
class GenericUri {
public:
    typedef typename ValueType::Ch Ch;  // 定义 Ch 为 ValueType::Ch 的别名
#if RAPIDJSON_HAS_STDSTRING
    typedef std::basic_string<Ch> String;  // 定义 String 为基于 Ch 类型的标准字符串
#endif

    //! Constructors
    GenericUri(Allocator* allocator = 0) : uri_(), base_(), scheme_(), auth_(), path_(), query_(), frag_(), allocator_(allocator), ownAllocator_() {
    }  // 构造函数，初始化各成员变量和分配器

    GenericUri(const Ch* uri, SizeType len, Allocator* allocator = 0) : uri_(), base_(), scheme_(), auth_(), path_(), query_(), frag_(), allocator_(allocator), ownAllocator_() {
        Parse(uri, len);  // 通过给定的 URI 和长度解析构造函数
    }

    GenericUri(const Ch* uri, Allocator* allocator = 0) : uri_(), base_(), scheme_(), auth_(), path_(), query_(), frag_(), allocator_(allocator), ownAllocator_() {
        Parse(uri, internal::StrLen<Ch>(uri));  // 通过给定的 URI 和长度解析构造函数
    }

    // Use with specializations of GenericValue
    # 定义一个模板类 GenericUri，接受一个类型 T 的参数 uri 和一个分配器 allocator
    GenericUri(const T& uri, Allocator* allocator = 0) : uri_(), base_(), scheme_(), auth_(), path_(), query_(), frag_(), allocator_(allocator), ownAllocator_() {
        # 从 uri 中获取类型为 Ch* 的指针 u
        const Ch* u = uri.template Get<const Ch*>(); // TypeHelper from document.h
        # 调用 Parse 方法，解析指针 u 指向的字符串，长度为字符串长度
        Parse(u, internal::StrLen<Ch>(u));
    }
#if RAPIDJSON_HAS_STDSTRING
    // 如果定义了 RAPIDJSON_HAS_STDSTRING 宏，则使用标准字符串构造函数
    GenericUri(const String& uri, Allocator* allocator = 0) : uri_(), base_(), scheme_(), auth_(), path_(), query_(), frag_(), allocator_(allocator), ownAllocator_() {
        // 解析传入的 URI 字符串
        Parse(uri.c_str(), internal::StrLen<Ch>(uri.c_str()));
    }
#endif

    //! 复制构造函数
    GenericUri(const GenericUri& rhs) : uri_(), base_(), scheme_(), auth_(), path_(), query_(), frag_(), allocator_(), ownAllocator_() {
        // 调用赋值运算符
        *this = rhs;
    }

    //! 复制构造函数
    GenericUri(const GenericUri& rhs, Allocator* allocator) : uri_(), base_(), scheme_(), auth_(), path_(), query_(), frag_(), allocator_(allocator), ownAllocator_() {
        // 调用赋值运算符
        *this = rhs;
    }

    //! 析构函数
    ~GenericUri() {
        // 释放资源
        Free();
        RAPIDJSON_DELETE(ownAllocator_);
    }

    //! 赋值运算符
    GenericUri& operator=(const GenericUri& rhs) {
        if (this != &rhs) {
            // 不删除 ownAllocator
            Free();
            // 分配内存
            Allocate(rhs.GetStringLength());
            // 复制各部分数据
            auth_ = CopyPart(scheme_, rhs.scheme_, rhs.GetSchemeStringLength());
            path_ = CopyPart(auth_, rhs.auth_, rhs.GetAuthStringLength());
            query_ = CopyPart(path_, rhs.path_, rhs.GetPathStringLength());
            frag_ = CopyPart(query_, rhs.query_, rhs.GetQueryStringLength());
            base_ = CopyPart(frag_, rhs.frag_, rhs.GetFragStringLength());
            uri_ = CopyPart(base_, rhs.base_, rhs.GetBaseStringLength());
            CopyPart(uri_, rhs.uri_, rhs.GetStringLength());
        }
        return *this;
    }

    //! 获取器
    // 与 GenericValue 的特化一起使用
    template<typename T> void Get(T& uri, Allocator& allocator) {
        uri.template Set<const Ch*>(this->GetString(), allocator); // TypeHelper from document.h
    }

    // 获取 URI 字符串
    const Ch* GetString() const { return uri_; }
    // 获取 URI 字符串长度
    SizeType GetStringLength() const { return uri_ == 0 ? 0 : internal::StrLen<Ch>(uri_); }
    // 获取基础 URI 字符串
    const Ch* GetBaseString() const { return base_; }
    // 返回基本字符串的长度，如果 base_ 为 0，则返回 0，否则返回 base_ 的长度
    SizeType GetBaseStringLength() const { return base_ == 0 ? 0 : internal::StrLen<Ch>(base_); }
    // 返回 scheme_ 字符串
    const Ch* GetSchemeString() const { return scheme_; }
    // 返回 scheme_ 字符串的长度，如果 scheme_ 为 0，则返回 0，否则返回 scheme_ 的长度
    SizeType GetSchemeStringLength() const { return scheme_ == 0 ? 0 : internal::StrLen<Ch>(scheme_); }
    // 返回 auth_ 字符串
    const Ch* GetAuthString() const { return auth_; }
    // 返回 auth_ 字符串的长度，如果 auth_ 为 0，则返回 0，否则返回 auth_ 的长度
    SizeType GetAuthStringLength() const { return auth_ == 0 ? 0 : internal::StrLen<Ch>(auth_); }
    // 返回 path_ 字符串
    const Ch* GetPathString() const { return path_; }
    // 返回 path_ 字符串的长度，如果 path_ 为 0，则返回 0，否则返回 path_ 的长度
    SizeType GetPathStringLength() const { return path_ == 0 ? 0 : internal::StrLen<Ch>(path_); }
    // 返回 query_ 字符串
    const Ch* GetQueryString() const { return query_; }
    // 返回 query_ 字符串的长度，如果 query_ 为 0，则返回 0，否则返回 query_ 的长度
    SizeType GetQueryStringLength() const { return query_ == 0 ? 0 : internal::StrLen<Ch>(query_); }
    // 返回 frag_ 字符串
    const Ch* GetFragString() const { return frag_; }
    // 返回 frag_ 字符串的长度，如果 frag_ 为 0，则返回 0，否则返回 frag_ 的长度
    SizeType GetFragStringLength() const { return frag_ == 0 ? 0 : internal::StrLen<Ch>(frag_); }
#if RAPIDJSON_HAS_STDSTRING
    // 如果定义了 RAPIDJSON_HAS_STDSTRING，则使用 String 类型返回 URI 的各个部分
    static String Get(const GenericUri& uri) { return String(uri.GetString(), uri.GetStringLength()); }
    static String GetBase(const GenericUri& uri) { return String(uri.GetBaseString(), uri.GetBaseStringLength()); }
    static String GetScheme(const GenericUri& uri) { return String(uri.GetSchemeString(), uri.GetSchemeStringLength()); }
    static String GetAuth(const GenericUri& uri) { return String(uri.GetAuthString(), uri.GetAuthStringLength()); }
    static String GetPath(const GenericUri& uri) { return String(uri.GetPathString(), uri.GetPathStringLength()); }
    static String GetQuery(const GenericUri& uri) { return String(uri.GetQueryString(), uri.GetQueryStringLength()); }
    static String GetFrag(const GenericUri& uri) { return String(uri.GetFragString(), uri.GetFragStringLength()); }
#endif

    //! Equality operators
    // 重载等于运算符，调用 Match 方法进行比较
    bool operator==(const GenericUri& rhs) const {
        return Match(rhs, true);
    }

    // 重载不等于运算符，调用 Match 方法进行比较
    bool operator!=(const GenericUri& rhs) const {
        return !Match(rhs, true);
    }

    // 比较两个 URI 是否匹配
    bool Match(const GenericUri& uri, bool full = true) const {
        Ch* s1;
        Ch* s2;
        if (full) {
            s1 = uri_;
            s2 = uri.uri_;
        } else {
            s1 = base_;
            s2 = uri.base_;
        }
        if (s1 == s2) return true;
        if (s1 == 0 || s2 == 0) return false;
        return internal::StrCmp<Ch>(s1, s2) == 0;
    }

    //! Resolve this URI against another (base) URI in accordance with URI resolution rules.
    // See https://tools.ietf.org/html/rfc3986
    // Use for resolving an id or $ref with an in-scope id.
    // Returns a new GenericUri for the resolved URI.
    // 根据 URI 解析规则解析当前 URI 和另一个（基本）URI，返回解析后的新 GenericUri

    //! Get the allocator of this GenericUri.
    // 返回当前 GenericUri 的分配器
    Allocator& GetAllocator() { return *allocator_; }

private:
    // Allocate memory for a URI
    // Returns total amount allocated
    // 为 URI 分配内存，返回总共分配的内存量
    // 分配内存空间
    std::size_t Allocate(std::size_t len) {
        // 如果用户没有提供分配器，则创建自己的分配器
        if (!allocator_)
            ownAllocator_ =  allocator_ = RAPIDJSON_NEW(Allocator)();

        // 分配一个包含 URI 的每个部分（5）加上基本部分和完整 URI 的块，全部以空字符结尾
        // 顺序：scheme, auth, path, query, frag, base, uri
        size_t total = (3 * len + 7) * sizeof(Ch);
        scheme_ = static_cast<Ch*>(allocator_->Malloc(total));
        *scheme_ = '\0';
        auth_ = scheme_ + 1;
        *auth_ = '\0';
        path_ = auth_ + 1;
        *path_ = '\0';
        query_ = path_ + 1;
        *query_ = '\0';
        frag_ = query_ + 1;
        *frag_ = '\0';
        base_ = frag_ + 1;
        *base_ = '\0';
        uri_ = base_ + 1;
        *uri_ = '\0';
        return total;
    }

    // 释放 URI 的内存
    void Free() {
        if (scheme_) {
            Allocator::Free(scheme_);
            scheme_ = 0;
        }
    }

    // 解析 URI 为组成部分：scheme, authority, path, query, & fragment
    // 支持与正则表达式 ^(([^:/?#]+):)?(//([^/?#]*))?([^?#]*)(\?([^#]*))?(#(.*))? 匹配的 URI，参考 https://tools.ietf.org/html/rfc3986
    }

    // 重建基本部分
    void SetBase() {
        Ch* next = base_;
        std::memcpy(next, scheme_, GetSchemeStringLength() * sizeof(Ch));
        next+= GetSchemeStringLength();
        std::memcpy(next, auth_, GetAuthStringLength() * sizeof(Ch));
        next+= GetAuthStringLength();
        std::memcpy(next, path_, GetPathStringLength() * sizeof(Ch));
        next+= GetPathStringLength();
        std::memcpy(next, query_, GetQueryStringLength() * sizeof(Ch));
        next+= GetQueryStringLength();
        *next = '\0';
    }

    // 重建 URI
    // 设置 URI
    void SetUri() {
        Ch* next = uri_; // 指向 URI 的指针
        std::memcpy(next, base_, GetBaseStringLength() * sizeof(Ch)); // 将 base_ 中的内容复制到 next 指向的位置
        next+= GetBaseStringLength(); // 移动 next 指针
        std::memcpy(next, frag_, GetFragStringLength() * sizeof(Ch)); // 将 frag_ 中的内容复制到 next 指向的位置
        next+= GetFragStringLength(); // 移动 next 指针
        *next = '\0'; // 在 next 指向的位置添加字符串结束符
    }

    // 复制一个 GenericUri 的一部分到另一个 GenericUri
    // 返回指向下一个要复制的部分的指针
    Ch* CopyPart(Ch* to, Ch* from, std::size_t len) {
        RAPIDJSON_ASSERT(to != 0); // 断言 to 不为空
        RAPIDJSON_ASSERT(from != 0); // 断言 from 不为空
        std::memcpy(to, from, len * sizeof(Ch)); // 将 from 中的内容复制到 to 指向的位置
        to[len] = '\0'; // 在 to 指向的位置添加字符串结束符
        Ch* next = to + len + 1; // 计算下一个要复制的部分的指针位置
        return next; // 返回下一个要复制的部分的指针
    }

    // 从 path_ 成员中移除 . 和 .. 段
    // https://tools.ietf.org/html/rfc3986
    // 这是在原地进行的，因为我们只是移除段
    }

    Ch* uri_;    // 整个 URI
    Ch* base_;   // 除了 fragment 之外的所有部分
    Ch* scheme_; // 包括 :
    Ch* auth_;   // 包括 //
    Ch* path_;   // 如果以 / 开头则为绝对路径
    Ch* query_;  // 包括 ?
    Ch* frag_;   // 包括 #

    Allocator* allocator_;      //!< 当前的分配器。它要么是用户提供的，要么等于 ownAllocator_。
    Allocator* ownAllocator_;   //!< 此 Uri 拥有的分配器。
// 结束 RapidJSON 命名空间
};

// 使用默认分配器和 UTF-8 编码创建 GenericUri 类型的 URI
typedef GenericUri<Value> Uri;

// 结束 RapidJSON 命名空间
RAPIDJSON_NAMESPACE_END

// 如果是 Clang 编译器，则恢复之前的诊断设置
#if defined(__clang__)
RAPIDJSON_DIAG_POP
#endif

// 结束条件编译，关闭 RapidJSON_URI_H_ 头文件
#endif // RAPIDJSON_URI_H_
```