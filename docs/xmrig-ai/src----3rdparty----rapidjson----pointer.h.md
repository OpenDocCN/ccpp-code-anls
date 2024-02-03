# `xmrig\src\3rdparty\rapidjson\pointer.h`

```cpp
// 如果 RAPIDJSON_POINTER_H_ 未定义，则定义 RAPIDJSON_POINTER_H_
#ifndef RAPIDJSON_POINTER_H_
#define RAPIDJSON_POINTER_H_

// 包含相关头文件
#include "document.h"
#include "uri.h"
#include "internal/itoa.h"

// 根据不同的编译器，关闭特定的警告
#ifdef __clang__
RAPIDJSON_DIAG_PUSH
RAPIDJSON_DIAG_OFF(switch-enum)
#elif defined(_MSC_VER)
RAPIDJSON_DIAG_PUSH
RAPIDJSON_DIAG_OFF(4512) // assignment operator could not be generated
#endif

// 命名空间的开始
RAPIDJSON_NAMESPACE_BEGIN

// 定义一个常量，表示无效的索引
static const SizeType kPointerInvalidIndex = ~SizeType(0);  //!< Represents an invalid index in GenericPointer::Token

// JSON Pointer 解析错误代码
enum PointerParseErrorCode {
    kPointerParseErrorNone = 0,                     //!< The parse is successful
    kPointerParseErrorTokenMustBeginWithSolidus,    //!< A token must begin with a '/'
    kPointerParseErrorInvalidEscape,                //!< Invalid escape
    kPointerParseErrorInvalidPercentEncoding,       //!< Invalid percent encoding in URI fragment
    kPointerParseErrorCharacterMustPercentEncode    //!< A character must percent encoded in URI fragment
};

///////////////////////////////////////////////////////////////////////////////
// GenericPointer

// 表示一个 JSON Pointer，使用 UTF8 编码和默认分配器
    # 这个类实现了 RFC 6901 "JavaScript Object Notation (JSON) Pointer" (https://tools.ietf.org/html/rfc6901)。
    
    # JSON 指针用于标识 JSON 文档（GenericDocument）中的特定值。它可以简化 DOM 树操作的编码，因为它可以通过单个 API 调用访问多层深度的 DOM 树。
    
    # 在将字符串表示（例如 "/foo/0"）或 URI 片段表示（例如 "#/foo/0"）解析为内部表示（标记）之后，它可以用于在多个文档或文档子树中解析特定值。
    
    # 与 GenericValue 不同，Pointer 可以进行复制构造和复制赋值。除了赋值之外，在构造后无法修改 Pointer。
    
    # 尽管 Pointer 非常方便，请注意构造 Pointer 涉及解析和动态内存分配。具有用户提供的标记的特殊构造函数可以消除这些问题。
    
    # GenericPointer 依赖于 GenericDocument 和 GenericValue。
    
    # \tparam ValueType DOM 树的值类型。例如 GenericValue<UTF8<> >
    # \tparam Allocator 用于为内部表示分配内存的分配器类型。
    
    # \note GenericPointer 使用与 ValueType 相同的编码。
    # 但是 GenericPointer 的分配器与 Value 的分配器是独立的。
*/
// 定义一个模板类 GenericPointer，用于表示 JSON 指针
template <typename ValueType, typename Allocator = CrtAllocator>
class GenericPointer {
public:
    // 从值中获取编码类型
    typedef typename ValueType::EncodingType EncodingType;  //!< 从值中获取编码类型
    // 从值中获取字符类型
    typedef typename ValueType::Ch Ch;                      //!< 从值中获取字符类型
    // 使用指定分配器的通用 URI 类型
    typedef GenericUri<ValueType, Allocator> UriType;


    //! 令牌是内部表示的基本单位
    /*!
        JSON 指针字符串表示 "/foo/123" 被解析为两个令牌：
        "foo" 和 123。123 将以数字形式和字符串形式表示。
        它们根据实际值类型（对象或数组）进行解析。

        对于不是数字的令牌，或者数字值超出范围
        （大于 SizeType 的限制），它们只被视为字符串形式
        （即令牌的索引将等于 kPointerInvalidIndex）。

        这个结构是公开的，以便用户可以创建一个指针，而无需解析和分配，使用特殊的构造函数。
    */
    struct Token {
        const Ch* name;             //!< 令牌的名称。它以空字符结尾，但可以包含空字符。
        SizeType length;            //!< 名称的长度。
        SizeType index;             //!< 有效的数组索引，如果不等于 kPointerInvalidIndex。
    };

    //!@name 构造函数和析构函数。
    //@{

    //! 默认构造函数。
    GenericPointer(Allocator* allocator = 0) : allocator_(allocator), ownAllocator_(), nameBuffer_(), tokens_(), tokenCount_(), parseErrorOffset_(), parseErrorCode_(kPointerParseErrorNone) {}

    //! 构造函数，解析字符串或 URI 片段表示。
    /*!
        \param source JSON 指针的以空字符结尾的字符串或 URI 片段表示。
        \param allocator 为此指针提供用户提供的分配器。如果未提供分配器，则创建一个自有的分配器。
    */
    # 定义 GenericPointer 类的构造函数，接受一个 const Ch* 类型的参数 source 和一个 Allocator* 类型的参数 allocator，默认为 0
    explicit GenericPointer(const Ch* source, Allocator* allocator = 0) : allocator_(allocator), ownAllocator_(), nameBuffer_(), tokens_(), tokenCount_(), parseErrorOffset_(), parseErrorCode_(kPointerParseErrorNone) {
        # 调用 Parse 函数，解析 source 指向的字符串，长度为 internal::StrLen(source)，并初始化成员变量
        Parse(source, internal::StrLen(source));
    }
#if RAPIDJSON_HAS_STDSTRING
    //! 如果定义了 RAPIDJSON_HAS_STDSTRING，则使用字符串或 URI 片段表示法来解析 JSON 指针的构造函数。
    /*!
        \param source JSON 指针的字符串或 URI 片段表示法。
        \param allocator 为该指针提供用户分配器。如果未提供分配器，则创建一个自有的分配器。
        \note 需要预处理符号 \ref RAPIDJSON_HAS_STDSTRING 的定义。
    */
    explicit GenericPointer(const std::basic_string<Ch>& source, Allocator* allocator = 0) : allocator_(allocator), ownAllocator_(), nameBuffer_(), tokens_(), tokenCount_(), parseErrorOffset_(), parseErrorCode_(kPointerParseErrorNone) {
        Parse(source.c_str(), source.size());
    }
#endif

    //! 使用字符串或 URI 片段表示法来解析 JSON 指针的构造函数，同时提供源字符串的长度。
    /*!
        \param source JSON 指针的字符串或 URI 片段表示法。
        \param length 源字符串的长度。
        \param allocator 为该指针提供用户分配器。如果未提供分配器，则创建一个自有的分配器。
        \note 比没有长度参数的重载略快。
    */
    GenericPointer(const Ch* source, size_t length, Allocator* allocator = 0) : allocator_(allocator), ownAllocator_(), nameBuffer_(), tokens_(), tokenCount_(), parseErrorOffset_(), parseErrorCode_(kPointerParseErrorNone) {
        Parse(source, length);
    }

    //! 使用用户提供的令牌进行构造的构造函数。
    /*!
        这个构造函数允许用户提供常量数组的令牌。
        这可以防止解析过程并消除分配。
        这在内存受限的环境中是首选。

        \param tokens 代表 JSON 指针的常量令牌数组。
        \param tokenCount 令牌数量。

        \b 示例
        \code
        #define NAME(s) { s, sizeof(s) / sizeof(s[0]) - 1, kPointerInvalidIndex }
        #define INDEX(i) { #i, sizeof(#i) - 1, i }

        static const Pointer::Token kTokens[] = { NAME("foo"), INDEX(123) };
        static const Pointer p(kTokens, sizeof(kTokens) / sizeof(kTokens[0]));
        // 等同于 static const Pointer p("/foo/123");

        #undef NAME
        #undef INDEX
        \endcode
    */
    GenericPointer(const Token* tokens, size_t tokenCount) : allocator_(), ownAllocator_(), nameBuffer_(), tokens_(const_cast<Token*>(tokens)), tokenCount_(tokenCount), parseErrorOffset_(), parseErrorCode_(kPointerParseErrorNone) {}

    //! 复制构造函数。
    GenericPointer(const GenericPointer& rhs) : allocator_(), ownAllocator_(), nameBuffer_(), tokens_(), tokenCount_(), parseErrorOffset_(), parseErrorCode_(kPointerParseErrorNone) {
        *this = rhs;
    }

    //! 复制构造函数。
    GenericPointer(const GenericPointer& rhs, Allocator* allocator) : allocator_(allocator), ownAllocator_(), nameBuffer_(), tokens_(), tokenCount_(), parseErrorOffset_(), parseErrorCode_(kPointerParseErrorNone) {
        *this = rhs;
    }

    //! 析构函数。
    ~GenericPointer() {
        if (nameBuffer_)    // 如果使用了用户提供的令牌构造函数，则 nameBuffer_ 为 nullptr，tokens_ 不会被释放。
            Allocator::Free(tokens_);
        RAPIDJSON_DELETE(ownAllocator_);
    }

    //! 赋值运算符。
    // 重载赋值运算符，将一个指针的内容赋值给另一个指针
    GenericPointer& operator=(const GenericPointer& rhs) {
        // 如果不是自己赋值给自己
        if (this != &rhs) {
            // 不删除自己的分配器
            if (nameBuffer_)
                Allocator::Free(tokens_);

            // 将右值的成员变量赋值给左值
            tokenCount_ = rhs.tokenCount_;
            parseErrorOffset_ = rhs.parseErrorOffset_;
            parseErrorCode_ = rhs.parseErrorCode_;

            // 如果右值有名称缓冲区，从中复制数据
            if (rhs.nameBuffer_)
                CopyFromRaw(rhs); // 通常是解析后的标记
            else {
                tokens_ = rhs.tokens_; // 用户提供的常量标记
                nameBuffer_ = 0;
            }
        }
        return *this;
    }

    //! 交换此指针的内容与另一个指针的内容
    /*!
        \param other 要交换的指针
        \note 常量复杂度
    */
    GenericPointer& Swap(GenericPointer& other) RAPIDJSON_NOEXCEPT {
        // 交换分配器
        internal::Swap(allocator_, other.allocator_);
        internal::Swap(ownAllocator_, other.ownAllocator_);
        internal::Swap(nameBuffer_, other.nameBuffer_);
        internal::Swap(tokens_, other.tokens_);
        internal::Swap(tokenCount_, other.tokenCount_);
        internal::Swap(parseErrorOffset_, other.parseErrorOffset_);
        internal::Swap(parseErrorCode_, other.parseErrorCode_);
        return *this;
    }

    //! 自由交换函数助手
    /*!
        通过 std::swap 实现常见交换模式的辅助函数：
        \code
        void swap(MyClass& a, MyClass& b) {
            using std::swap;
            swap(a.pointer, b.pointer);
            // ...
        }
        \endcode
        \see Swap()
     */
    friend inline void swap(GenericPointer& a, GenericPointer& b) RAPIDJSON_NOEXCEPT { a.Swap(b); }

    //@}

    //!@name 追加标记
    //@{

    //! 追加一个标记并返回一个新的指针
    /*!
        \param token 要追加的标记
        \param allocator 用于新返回的指针的分配器
        \return 追加了标记的新指针
    */
    /*
    在当前指针对象上追加一个名字标记，并返回一个新的指针对象
    */
    GenericPointer Append(const Token& token, Allocator* allocator = 0) const {
        GenericPointer r;
        r.allocator_ = allocator;
        Ch *p = r.CopyFromRaw(*this, 1, token.length + 1);
        std::memcpy(p, token.name, (token.length + 1) * sizeof(Ch));
        r.tokens_[tokenCount_].name = p;
        r.tokens_[tokenCount_].length = token.length;
        r.tokens_[tokenCount_].index = token.index;
        return r;
    }

    //! 追加一个具有长度的名字标记，并返回一个新的指针
    /*!
        \param name 要追加的名字
        \param length 名字的长度
        \param allocator 用于新返回的指针的分配器
        \return 一个追加了标记的新指针
    */
    GenericPointer Append(const Ch* name, SizeType length, Allocator* allocator = 0) const {
        Token token = { name, length, kPointerInvalidIndex };
        return Append(token, allocator);
    }

    //! 追加一个没有长度的名字标记，并返回一个新的指针
    /*!
        \param name 要追加的名字（const Ch*）
        \param allocator 用于新返回的指针的分配器
        \return 一个追加了标记的新指针
    */
    template <typename T>
    RAPIDJSON_DISABLEIF_RETURN((internal::NotExpr<internal::IsSame<typename internal::RemoveConst<T>::Type, Ch> >), (GenericPointer))
    Append(T* name, Allocator* allocator = 0) const {
        return Append(name, internal::StrLen(name), allocator);
    }
#if RAPIDJSON_HAS_STDSTRING
    //! 如果编译器支持 std::string，则追加一个名称标记，并返回一个新的指针
    /*!
        \param name 要追加的名称。
        \param allocator 用于新返回的指针的分配器。
        \return 一个追加了标记的新指针。
    */
    GenericPointer Append(const std::basic_string<Ch>& name, Allocator* allocator = 0) const {
        return Append(name.c_str(), static_cast<SizeType>(name.size()), allocator);
    }
#endif

    //! 追加一个索引标记，并返回一个新的指针
    /*!
        \param index 要追加的索引。
        \param allocator 用于新返回的指针的分配器。
        \return 一个追加了标记的新指针。
    */
    GenericPointer Append(SizeType index, Allocator* allocator = 0) const {
        char buffer[21];
        char* end = sizeof(SizeType) == 4 ? internal::u32toa(index, buffer) : internal::u64toa(index, buffer);
        SizeType length = static_cast<SizeType>(end - buffer);
        buffer[length] = '\0';

        if (sizeof(Ch) == 1) {
            Token token = { reinterpret_cast<Ch*>(buffer), length, index };
            return Append(token, allocator);
        }
        else {
            Ch name[21];
            for (size_t i = 0; i <= length; i++)
                name[i] = static_cast<Ch>(buffer[i]);
            Token token = { name, length, index };
            return Append(token, allocator);
        }
    }

    //! 通过值追加一个标记，并返回一个新的指针
    /*!
        \param token 要追加的标记。
        \param allocator 用于新返回的指针的分配器。
        \return 一个追加了标记的新指针。
    */
    // 在当前指针后追加一个值，可选参数为分配器
    GenericPointer Append(const ValueType& token, Allocator* allocator = 0) const {
        // 如果 token 是字符串类型，则调用 Append 函数追加字符串
        if (token.IsString())
            return Append(token.GetString(), token.GetStringLength(), allocator);
        // 否则，断言 token 是无符号整数类型，并且值小于等于 SizeType(~0)
        else {
            RAPIDJSON_ASSERT(token.IsUint64());
            RAPIDJSON_ASSERT(token.GetUint64() <= SizeType(~0));
            return Append(static_cast<SizeType>(token.GetUint64()), allocator);
        }
    }

    //!@name 处理解析错误
    //@{

    //! 检查当前指针是否有效
    bool IsValid() const { return parseErrorCode_ == kPointerParseErrorNone; }

    //! 获取解析错误的偏移量（以代码单元为单位）
    size_t GetParseErrorOffset() const { return parseErrorOffset_; }

    //! 获取解析错误代码
    PointerParseErrorCode GetParseErrorCode() const { return parseErrorCode_; }

    //@}

    //! 获取当前指针的分配器
    Allocator& GetAllocator() { return *allocator_; }

    //!@name Tokens
    //@{

    //! 获取令牌数组（只读版本）
    const Token* GetTokens() const { return tokens_; }

    //! 获取令牌数量
    size_t GetTokenCount() const { return tokenCount_; }

    //@}

    //!@name 相等/不相等运算符
    //@{

    //! 相等运算符
    /*!
        \note 当任何指针无效时，始终返回 false
    */
    bool operator==(const GenericPointer& rhs) const {
        if (!IsValid() || !rhs.IsValid() || tokenCount_ != rhs.tokenCount_)
            return false;

        for (size_t i = 0; i < tokenCount_; i++) {
            if (tokens_[i].index != rhs.tokens_[i].index ||
                tokens_[i].length != rhs.tokens_[i].length ||
                (tokens_[i].length != 0 && std::memcmp(tokens_[i].name, rhs.tokens_[i].name, sizeof(Ch)* tokens_[i].length) != 0))
            {
                return false;
            }
        }

        return true;
    }

    //! 不相等运算符
    /*!
        \note 当任何指针无效时，始终返回 true
    */
    bool operator!=(const GenericPointer& rhs) const {
        return !(*this == rhs);
    }

    //@}
    */
    // 重载不等于运算符
    bool operator!=(const GenericPointer& rhs) const { return !(*this == rhs); }

    //! Less than operator.
    /*!
        \note Invalid pointers are always greater than valid ones.
    */
    // 重载小于运算符
    bool operator<(const GenericPointer& rhs) const {
        // 如果当前指针无效，则返回false
        if (!IsValid())
            return false;
        // 如果rhs指针无效，则返回true
        if (!rhs.IsValid())
            return true;

        // 比较tokenCount_
        if (tokenCount_ != rhs.tokenCount_)
            return tokenCount_ < rhs.tokenCount_;

        // 遍历tokens_数组
        for (size_t i = 0; i < tokenCount_; i++) {
            // 比较index
            if (tokens_[i].index != rhs.tokens_[i].index)
                return tokens_[i].index < rhs.tokens_[i].index;

            // 比较length
            if (tokens_[i].length != rhs.tokens_[i].length)
                return tokens_[i].length < rhs.tokens_[i].length;

            // 比较name
            if (int cmp = std::memcmp(tokens_[i].name, rhs.tokens_[i].name, sizeof(Ch) * tokens_[i].length))
                return cmp < 0;
        }

        return false;
    }

    //@}

    //!@name Stringify
    //@{

    //! Stringify the pointer into string representation.
    /*!
        \tparam OutputStream Type of output stream.
        \param os The output stream.
    */
    // 将指针转换为字符串表示形式
    template<typename OutputStream>
    bool Stringify(OutputStream& os) const {
        return Stringify<false, OutputStream>(os);
    }

    //! Stringify the pointer into URI fragment representation.
    /*!
        \tparam OutputStream Type of output stream.
        \param os The output stream.
    */
    // 将指针转换为URI片段表示形式
    template<typename OutputStream>
    bool StringifyUriFragment(OutputStream& os) const {
        return Stringify<true, OutputStream>(os);
    }

    //@}

    //!@name Create value
    //@{

    //! Create a value in a subtree.
    /*!
        如果值不存在，则创建所有父级值和一个 JSON 空值。
        因此它总是成功的，并返回新创建的或现有的值。

        请注意，根据标记，它可能会更改父级的类型，因此它可能会删除先前存储的值。例如，如果文档是一个数组，并且使用 "/foo" 创建一个值，那么文档将被更改为对象，并且所有现有的数组元素都将丢失。

        \param root 要解析的 DOM 子树的根值。它可以是文档根之外的任何值。
        \param allocator 用于创建值的分配器，如果指定的值或其父级不存在。
        \param alreadyExist 如果非空，则存储解析的值是否已经存在。
        \return 已解析的新创建的（一个 JSON 空值），或已存在的值。
    */
    }

    //! 在文档中创建一个值。
    /*!
        \param document 要解析的文档。
        \param alreadyExist 如果非空，则存储解析的值是否已经存在。
        \return 已解析的新创建的，或已存在的值。
    */
    template <typename stackAllocator>
    ValueType& Create(GenericDocument<EncodingType, typename ValueType::AllocatorType, stackAllocator>& document, bool* alreadyExist = 0) const {
        return Create(document, document.GetAllocator(), alreadyExist);
    }

    //@}

    //!@name 计算 URI
    //@{

    //! 计算子树的作用域 URI。
    //  用于 JSON 指针到 JSON 模式文档。
    /*!
        \param root Root value of a DOM sub-tree to be resolved. It can be any value other than document root.
        \param rootUri Root URI
        \param unresolvedTokenIndex If the pointer cannot resolve a token in the pointer, this parameter can obtain the index of unresolved token.
        \param allocator Allocator for Uris
        \return Uri if it can be resolved. Otherwise null.
    
        \note
        There are only 3 situations when a URI cannot be resolved:
        1. A value in the path is not an array nor object.
        2. An object value does not contain the token.
        3. A token is out of range of an array value.
    
        Use unresolvedTokenIndex to retrieve the token index.
    */
    // 根据给定的根值和根 URI 获取 URI 类型的值
    UriType GetUri(ValueType& root, const UriType& rootUri, size_t* unresolvedTokenIndex = 0, Allocator* allocator = 0) const {
        // 定义 ID 字符串和对应的值
        static const Ch kIdString[] = { 'i', 'd', '\0' };
        static const ValueType kIdValue(kIdString, 2);
        // 创建基础 URI 对象
        UriType base = UriType(rootUri, allocator);
        // 断言根值有效
        RAPIDJSON_ASSERT(IsValid());
        // 获取根值的指针
        ValueType* v = &root;
        // 遍历令牌数组
        for (const Token *t = tokens_; t != tokens_ + tokenCount_; ++t) {
            switch (v->GetType()) {
                case kObjectType:
                {
                    // 查找是否有 ID，并且如果有的话，使用当前基础 URI 解析
                    typename ValueType::MemberIterator m = v->FindMember(kIdValue);
                    if (m != v->MemberEnd() && (m->value).IsString()) {
                        // 解析当前 URI
                        UriType here = UriType(m->value, allocator).Resolve(base, allocator);
                        base = here;
                    }
                    // 查找当前令牌对应的成员
                    m = v->FindMember(GenericValue<EncodingType>(GenericStringRef<Ch>(t->name, t->length)));
                    if (m == v->MemberEnd())
                        break;
                    v = &m->value;
                }
                  continue;
                case kArrayType:
                    // 如果索引无效或者超出数组大小，则跳过
                    if (t->index == kPointerInvalidIndex || t->index >= v->Size())
                        break;
                    v = &((*v)[t->index]);
                    continue;
                default:
                    break;
            }

            // 错误：未解析的令牌
            if (unresolvedTokenIndex)
                *unresolvedTokenIndex = static_cast<size_t>(t - tokens_);
            return UriType(allocator);
        }
        return base;
    }

    // 获取给定根值和根 URI 的 URI 类型的值
    UriType GetUri(const ValueType& root, const UriType& rootUri, size_t* unresolvedTokenIndex = 0, Allocator* allocator = 0) const {
      return GetUri(const_cast<ValueType&>(root), rootUri, unresolvedTokenIndex, allocator);
    }


    //!@name Query value
    //@{
    //! 在子树中查询一个值。
    /*!
        \param root 要解析的 DOM 子树的根值。它可以是文档根之外的任何值。
        \param unresolvedTokenIndex 如果指针无法解析指针中的令牌，则此参数可以获取未解析令牌的索引。
        \return 如果可以解析值，则返回指向该值的指针。否则返回空指针。

        \note
        只有 3 种情况下无法解析值：
        1. 路径中的值既不是数组也不是对象。
        2. 对象值不包含该令牌。
        3. 令牌超出了数组值的范围。

        使用 unresolvedTokenIndex 来检索令牌索引。
    */
    ValueType* Get(ValueType& root, size_t* unresolvedTokenIndex = 0) const {
        RAPIDJSON_ASSERT(IsValid());
        ValueType* v = &root;
        for (const Token *t = tokens_; t != tokens_ + tokenCount_; ++t) {
            switch (v->GetType()) {
            case kObjectType:
                {
                    // 如果值是对象类型，则查找成员
                    typename ValueType::MemberIterator m = v->FindMember(GenericValue<EncodingType>(GenericStringRef<Ch>(t->name, t->length)));
                    if (m == v->MemberEnd())
                        break;
                    v = &m->value;
                }
                continue;
            case kArrayType:
                // 如果值是数组类型，则检查索引是否有效
                if (t->index == kPointerInvalidIndex || t->index >= v->Size())
                    break;
                v = &((*v)[t->index]);
                continue;
            default:
                break;
            }

            // 错误：未解析的令牌
            if (unresolvedTokenIndex)
                *unresolvedTokenIndex = static_cast<size_t>(t - tokens_);
            return 0;
        }
        return v;
    }

    //! 在常量子树中查询一个常量值。
    /*!
        \param root 根节点的值，用于解析 DOM 子树。它可以是文档根之外的任何值。
        \return 如果可以解析，则返回指向该值的指针。否则返回空指针。
    */
    const ValueType* Get(const ValueType& root, size_t* unresolvedTokenIndex = 0) const {
        return Get(const_cast<ValueType&>(root), unresolvedTokenIndex);
    }

    //@}

    //!@name Query a value with default
    //@{

    //! 使用默认值查询子树中的值。
    /*!
        类似于 Get()，但如果指定的值不存在，则创建所有父级并克隆默认值。
        因此，此函数始终成功。

        \param root 根节点的值，用于解析 DOM 子树。它可以是文档根之外的任何值。
        \param defaultValue 如果值不存在，则克隆的默认值。
        \param allocator 用于创建值的分配器，如果指定的值或其父级不存在。
        \see Create()
    */
    ValueType& GetWithDefault(ValueType& root, const ValueType& defaultValue, typename ValueType::AllocatorType& allocator) const {
        bool alreadyExist;
        ValueType& v = Create(root, allocator, &alreadyExist);
        return alreadyExist ? v : v.CopyFrom(defaultValue, allocator);
    }

    //! 使用默认的空结尾字符串查询子树中的值。
    ValueType& GetWithDefault(ValueType& root, const Ch* defaultValue, typename ValueType::AllocatorType& allocator) const {
        bool alreadyExist;
        ValueType& v = Create(root, allocator, &alreadyExist);
        return alreadyExist ? v : v.SetString(defaultValue, allocator);
    }
#if RAPIDJSON_HAS_STDSTRING
    //! 如果支持标准字符串，则查询具有默认 std::basic_string 的子树中的值。
    ValueType& GetWithDefault(ValueType& root, const std::basic_string<Ch>& defaultValue, typename ValueType::AllocatorType& allocator) const {
        // 创建一个值，如果已经存在则返回已存在的值，否则返回默认值
        bool alreadyExist;
        ValueType& v = Create(root, allocator, &alreadyExist);
        return alreadyExist ? v : v.SetString(defaultValue, allocator);
    }
#endif

    //! 查询具有默认原始值的子树中的值。
    /*!
        \tparam T 要么是 Type，int，unsigned，int64_t，uint64_t，bool
    */
    template <typename T>
    RAPIDJSON_DISABLEIF_RETURN((internal::OrExpr<internal::IsPointer<T>, internal::IsGenericValue<T> >), (ValueType&))
    GetWithDefault(ValueType& root, T defaultValue, typename ValueType::AllocatorType& allocator) const {
        return GetWithDefault(root, ValueType(defaultValue).Move(), allocator);
    }

    //! 查询具有默认值的文档中的值。
    template <typename stackAllocator>
    ValueType& GetWithDefault(GenericDocument<EncodingType, typename ValueType::AllocatorType, stackAllocator>& document, const ValueType& defaultValue) const {
        return GetWithDefault(document, defaultValue, document.GetAllocator());
    }

    //! 查询具有默认空终止字符串的文档中的值。
    template <typename stackAllocator>
    ValueType& GetWithDefault(GenericDocument<EncodingType, typename ValueType::AllocatorType, stackAllocator>& document, const Ch* defaultValue) const {
        return GetWithDefault(document, defaultValue, document.GetAllocator());
    }

#if RAPIDJSON_HAS_STDSTRING
    //! 查询具有默认 std::basic_string 的文档中的值。
    template <typename stackAllocator>
    # 从 JSON 文档中获取指定键对应的值，如果键不存在则返回默认值
    ValueType& GetWithDefault(GenericDocument<EncodingType, typename ValueType::AllocatorType, stackAllocator>& document, const std::basic_string<Ch>& defaultValue) const {
        # 调用另一个重载的 GetWithDefault 函数，传入 JSON 文档、默认值和 JSON 文档的分配器
        return GetWithDefault(document, defaultValue, document.GetAllocator());
    }
#endif

//! Query a value in a document with default primitive value.
/*!
    \tparam T Either \ref Type, \c int, \c unsigned, \c int64_t, \c uint64_t, \c bool
*/
template <typename T, typename stackAllocator>
RAPIDJSON_DISABLEIF_RETURN((internal::OrExpr<internal::IsPointer<T>, internal::IsGenericValue<T> >), (ValueType&))
GetWithDefault(GenericDocument<EncodingType, typename ValueType::AllocatorType, stackAllocator>& document, T defaultValue) const {
    return GetWithDefault(document, defaultValue, document.GetAllocator());
}

//@}

//!@name Set a value
//@{

//! Set a value in a subtree, with move semantics.
/*!
    It creates all parents if they are not exist or types are different to the tokens.
    So this function always succeeds but potentially remove existing values.

    \param root Root value of a DOM sub-tree to be resolved. It can be any value other than document root.
    \param value Value to be set.
    \param allocator Allocator for creating the values if the specified value or its parents are not exist.
    \see Create()
*/
ValueType& Set(ValueType& root, ValueType& value, typename ValueType::AllocatorType& allocator) const {
    return Create(root, allocator) = value;
}

//! Set a value in a subtree, with copy semantics.
ValueType& Set(ValueType& root, const ValueType& value, typename ValueType::AllocatorType& allocator) const {
    return Create(root, allocator).CopyFrom(value, allocator);
}

//! Set a null-terminated string in a subtree.
ValueType& Set(ValueType& root, const Ch* value, typename ValueType::AllocatorType& allocator) const {
    return Create(root, allocator) = ValueType(value, allocator).Move();
}

#if RAPIDJSON_HAS_STDSTRING
//! Set a std::basic_string in a subtree.
    # 设置给定根节点的值
    ValueType& Set(ValueType& root, const std::basic_string<Ch>& value, typename ValueType::AllocatorType& allocator) const {
        # 创建一个新的值，并将其赋给根节点
        return Create(root, allocator) = ValueType(value, allocator).Move();
    }
#endif

//! 在子树中设置一个基本值。
/*!
    \tparam T 要么是 \ref Type，要么是 \c int，\c unsigned，\c int64_t，\c uint64_t，\c bool
*/
template <typename T>
RAPIDJSON_DISABLEIF_RETURN((internal::OrExpr<internal::IsPointer<T>, internal::IsGenericValue<T> >), (ValueType&))
Set(ValueType& root, T value, typename ValueType::AllocatorType& allocator) const {
    return Create(root, allocator) = ValueType(value).Move();
}

//! 使用移动语义在文档中设置一个值。
template <typename stackAllocator>
ValueType& Set(GenericDocument<EncodingType, typename ValueType::AllocatorType, stackAllocator>& document, ValueType& value) const {
    return Create(document) = value;
}

//! 使用复制语义在文档中设置一个值。
template <typename stackAllocator>
ValueType& Set(GenericDocument<EncodingType, typename ValueType::AllocatorType, stackAllocator>& document, const ValueType& value) const {
    return Create(document).CopyFrom(value, document.GetAllocator());
}

//! 在文档中设置一个以空字符结尾的字符串。
template <typename stackAllocator>
ValueType& Set(GenericDocument<EncodingType, typename ValueType::AllocatorType, stackAllocator>& document, const Ch* value) const {
    return Create(document) = ValueType(value, document.GetAllocator()).Move();
}

#if RAPIDJSON_HAS_STDSTRING
//! 在文档中设置一个 std::basic_string。
template <typename stackAllocator>
ValueType& Set(GenericDocument<EncodingType, typename ValueType::AllocatorType, stackAllocator>& document, const std::basic_string<Ch>& value) const {
    return Create(document) = ValueType(value, document.GetAllocator()).Move();
}
#endif

//! 在文档中设置一个基本值。
/*!
\tparam T 要么是 \ref Type，要么是 \c int，\c unsigned，\c int64_t，\c uint64_t，\c bool
*/
template <typename T, typename stackAllocator>
    # 禁用条件返回，如果 T 是指针或者通用值，则返回 ValueType 的引用
    RAPIDJSON_DISABLEIF_RETURN((internal::OrExpr<internal::IsPointer<T>, internal::IsGenericValue<T> >), (ValueType&))
        Set(GenericDocument<EncodingType, typename ValueType::AllocatorType, stackAllocator>& document, T value) const {
            return Create(document) = value;
    }

    //!@name Swap a value
    //@{

    //! 与子树中的值交换一个值
    /*!
        如果不存在或类型与标记不同，则创建所有父级。因此，此函数始终成功，但可能会删除现有值。

        \param root 要解析的 DOM 子树的根值。它可以是文档根之外的任何值。
        \param value 要交换的值。
        \param allocator 用于创建值的分配器，如果指定的值或其父级不存在。
        \see Create()
    */
    ValueType& Swap(ValueType& root, ValueType& value, typename ValueType::AllocatorType& allocator) const {
        return Create(root, allocator).Swap(value);
    }

    //! 与文档中的值交换一个值。
    template <typename stackAllocator>
    ValueType& Swap(GenericDocument<EncodingType, typename ValueType::AllocatorType, stackAllocator>& document, ValueType& value) const {
        return Create(document).Swap(value);
    }

    //@}

    //! 从子树中删除一个值。
    /*!
        \param root 要解析的 DOM 子树的根值。它可以是文档根之外的任何值。
        \return 是否找到并删除了解析的值。

        \note 使用空指针 \c Pointer("") 进行擦除，即根，总是失败并返回 false。
    */
    // 擦除 JSON 对象中的指定节点
    bool Erase(ValueType& root) const {
        // 断言 JSON 对象有效
        RAPIDJSON_ASSERT(IsValid());
        // 如果 tokenCount 为 0，则无法擦除根节点
        if (tokenCount_ == 0) 
            return false;

        // 初始化指向根节点的指针
        ValueType* v = &root;
        // 获取最后一个 token
        const Token* last = tokens_ + (tokenCount_ - 1);
        // 遍历 token 数组
        for (const Token *t = tokens_; t != last; ++t) {
            // 根据节点类型进行处理
            switch (v->GetType()) {
            case kObjectType:
                {
                    // 在对象中查找指定成员
                    typename ValueType::MemberIterator m = v->FindMember(GenericValue<EncodingType>(GenericStringRef<Ch>(t->name, t->length)));
                    // 如果未找到指定成员，则返回 false
                    if (m == v->MemberEnd())
                        return false;
                    // 更新指针指向找到的成员值
                    v = &m->value;
                }
                break;
            case kArrayType:
                // 如果索引无效或超出范围，则返回 false
                if (t->index == kPointerInvalidIndex || t->index >= v->Size())
                    return false;
                // 更新指针指向指定索引的数组元素
                v = &((*v)[t->index]);
                break;
            default:
                return false;
            }
        }

        // 根据最后一个 token 的类型进行处理
        switch (v->GetType()) {
        case kObjectType:
            // 擦除对象中的指定成员
            return v->EraseMember(GenericStringRef<Ch>(last->name, last->length));
        case kArrayType:
            // 如果索引无效或超出范围，则返回 false
            if (last->index == kPointerInvalidIndex || last->index >= v->Size())
                return false;
            // 擦除数组中指定索引的元素
            v->Erase(v->Begin() + last->index);
            return true;
        default:
            return false;
        }
    }
private:
    //! Clone the content from rhs to this.
    /*!
        \param rhs Source pointer.
        \param extraToken Extra tokens to be allocated.
        \param extraNameBufferSize Extra name buffer size (in number of Ch) to be allocated.
        \return Start of non-occupied name buffer, for storing extra names.
    */
    Ch* CopyFromRaw(const GenericPointer& rhs, size_t extraToken = 0, size_t extraNameBufferSize = 0) {
        // 如果 allocator_ 为空，则独立拥有 allocator
        if (!allocator_) // allocator is independently owned.
            ownAllocator_ = allocator_ = RAPIDJSON_NEW(Allocator)();

        // 计算需要分配的名字缓冲区大小
        size_t nameBufferSize = rhs.tokenCount_; // null terminators for tokens
        for (Token *t = rhs.tokens_; t != rhs.tokens_ + rhs.tokenCount_; ++t)
            nameBufferSize += t->length;

        // 分配内存并复制内容
        tokenCount_ = rhs.tokenCount_ + extraToken;
        tokens_ = static_cast<Token *>(allocator_->Malloc(tokenCount_ * sizeof(Token) + (nameBufferSize + extraNameBufferSize) * sizeof(Ch)));
        nameBuffer_ = reinterpret_cast<Ch *>(tokens_ + tokenCount_);
        if (rhs.tokenCount_ > 0) {
            std::memcpy(tokens_, rhs.tokens_, rhs.tokenCount_ * sizeof(Token));
        }
        if (nameBufferSize > 0) {
            std::memcpy(nameBuffer_, rhs.nameBuffer_, nameBufferSize * sizeof(Ch));
        }

        // 调整指向名字缓冲区的指针
        std::ptrdiff_t diff = nameBuffer_ - rhs.nameBuffer_;
        for (Token *t = tokens_; t != tokens_ + rhs.tokenCount_; ++t)
            t->name += diff;

        return nameBuffer_ + nameBufferSize;
    }

    //! Check whether a character should be percent-encoded.
    /*!
        According to RFC 3986 2.3 Unreserved Characters.
        \param c The character (code unit) to be tested.
    */
    bool NeedPercentEncode(Ch c) const {
        // 检查字符是否需要进行百分号编码
        return !((c >= '0' && c <= '9') || (c >= 'A' && c <='Z') || (c >= 'a' && c <= 'z') || c == '-' || c == '.' || c == '_' || c =='~');
    }

    //! Parse a JSON String or its URI fragment representation into tokens.
#ifndef __clang__ // -Wdocumentation
    /*!
        \param source Either a JSON Pointer string, or its URI fragment representation. Not need to be null terminated.
        \param length Length of the source string.
        \note Source cannot be JSON String Representation of JSON Pointer, e.g. In "/\u0000", \u0000 will not be unescaped.
    */
#endif
    // 释放内存并重置相关变量，然后返回
    error:
        Allocator::Free(tokens_);
        nameBuffer_ = 0;
        tokens_ = 0;
        tokenCount_ = 0;
        parseErrorOffset_ = i;
        return;
    }

    //! Stringify to string or URI fragment representation.
    /*!
        \tparam uriFragment True for stringifying to URI fragment representation. False for string representation.
        \tparam OutputStream type of output stream.
        \param os The output stream.
    */
    // 将 JSON Pointer 转换为字符串或 URI 片段表示
    template<bool uriFragment, typename OutputStream>
    bool Stringify(OutputStream& os) const {
        RAPIDJSON_ASSERT(IsValid());

        if (uriFragment)
            os.Put('#');

        for (Token *t = tokens_; t != tokens_ + tokenCount_; ++t) {
            os.Put('/');
            for (size_t j = 0; j < t->length; j++) {
                Ch c = t->name[j];
                if (c == '~') {
                    os.Put('~');
                    os.Put('0');
                }
                else if (c == '/') {
                    os.Put('~');
                    os.Put('1');
                }
                else if (uriFragment && NeedPercentEncode(c)) {
                    // 转换为 UTF8 序列
                    GenericStringStream<typename ValueType::EncodingType> source(&t->name[j]);
                    PercentEncodeStream<OutputStream> target(os);
                    if (!Transcoder<EncodingType, UTF8<> >().Validate(source, target))
                        return false;
                    j += source.Tell() - 1;
                }
                else
                    os.Put(c);
            }
        }
        return true;
    }
    //! 用于将百分号编码序列解码为代码单元的辅助流。
    /*!
        该流将 %XY 三元组解码为代码单元（0-255）。
        如果遇到无效字符，它将将输出代码单元设置为0，并标记为无效，需要通过 IsValid() 进行检查。
    */
    class PercentDecodeStream {
    public:
        typedef typename ValueType::Ch Ch;

        //! 构造函数
        /*!
            \param source 流的起始位置
            \param end 流的结束位置的下一个位置。
        */
        PercentDecodeStream(const Ch* source, const Ch* end) : src_(source), head_(source), end_(end), valid_(true) {}

        Ch Take() {
            if (*src_ != '%' || src_ + 3 > end_) { // %XY 三元组
                valid_ = false;
                return 0;
            }
            src_++;
            Ch c = 0;
            for (int j = 0; j < 2; j++) {
                c = static_cast<Ch>(c << 4);
                Ch h = *src_;
                if      (h >= '0' && h <= '9') c = static_cast<Ch>(c + h - '0');
                else if (h >= 'A' && h <= 'F') c = static_cast<Ch>(c + h - 'A' + 10);
                else if (h >= 'a' && h <= 'f') c = static_cast<Ch>(c + h - 'a' + 10);
                else {
                    valid_ = false;
                    return 0;
                }
                src_++;
            }
            return c;
        }

        size_t Tell() const { return static_cast<size_t>(src_ - head_); }
        bool IsValid() const { return valid_; }

    private:
        const Ch* src_;     //!< 当前读取位置。
        const Ch* head_;    //!< 字符串的原始起始位置。
        const Ch* end_;     //!< 结束位置的下一个位置。
        bool valid_;        //!< 解析是否有效的标志。
    };

    //! 用于将字符（UTF-8代码单元）编码为百分号编码序列的辅助流。
    template <typename OutputStream>
    class PercentEncodeStream {
    // 定义一个公有类成员函数 PercentEncodeStream，接受一个 OutputStream 的引用作为参数
    public:
        PercentEncodeStream(OutputStream& os) : os_(os) {}
        // 定义 Put 函数，用于将字符进行百分号编码并输出到 os_ 中
        void Put(char c) { // UTF-8 must be byte
            // 将字符转换为无符号字符
            unsigned char u = static_cast<unsigned char>(c);
            // 定义十六进制数字的数组
            static const char hexDigits[16] = { '0', '1', '2', '3', '4', '5', '6', '7', '8', '9', 'A', 'B', 'C', 'D', 'E', 'F' };
            // 输出 %
            os_.Put('%');
            // 输出字符的高四位对应的十六进制数字
            os_.Put(static_cast<typename OutputStream::Ch>(hexDigits[u >> 4]));
            // 输出字符的低四位对应的十六进制数字
            os_.Put(static_cast<typename OutputStream::Ch>(hexDigits[u & 15]));
        }
    // 定义一个私有类成员变量 os_，用于存储 OutputStream 的引用
    private:
        OutputStream& os_;
    };

    // 当前分配器，可能是用户提供的，也可能是 ownAllocator_
    Allocator* allocator_;                  
    // Pointer 拥有的分配器
    Allocator* ownAllocator_;               
    // 包含所有标记名称的缓冲区
    Ch* nameBuffer_;                        
    // 一个标记列表
    Token* tokens_;                         
    // tokens_ 中的标记数量
    size_t tokenCount_;                     
    // 解析失败时的代码单元偏移量
    size_t parseErrorOffset_;               
    // 解析错误代码
    PointerParseErrorCode parseErrorCode_;  
// 结束了前面的代码块
};

// 为值（UTF-8，默认分配器）创建通用指针
typedef GenericPointer<Value> Pointer;

// 辅助函数，用于为通用指针创建值
template <typename T>
typename T::ValueType& CreateValueByPointer(T& root, const GenericPointer<typename T::ValueType>& pointer, typename T::AllocatorType& a) {
    return pointer.Create(root, a);
}

// 辅助函数，用于为通用指针创建值
template <typename T, typename CharType, size_t N>
typename T::ValueType& CreateValueByPointer(T& root, const CharType(&source)[N], typename T::AllocatorType& a) {
    return GenericPointer<typename T::ValueType>(source, N - 1).Create(root, a);
}

// 无分配器参数的辅助函数
template <typename DocumentType>
typename DocumentType::ValueType& CreateValueByPointer(DocumentType& document, const GenericPointer<typename DocumentType::ValueType>& pointer) {
    return pointer.Create(document);
}

// 无分配器参数的辅助函数
template <typename DocumentType, typename CharType, size_t N>
typename DocumentType::ValueType& CreateValueByPointer(DocumentType& document, const CharType(&source)[N]) {
    return GenericPointer<typename DocumentType::ValueType>(source, N - 1).Create(document);
}

// 辅助函数，用于通过通用指针获取值
template <typename T>
typename T::ValueType* GetValueByPointer(T& root, const GenericPointer<typename T::ValueType>& pointer, size_t* unresolvedTokenIndex = 0) {
    return pointer.Get(root, unresolvedTokenIndex);
}

// 辅助函数，用于通过通用指针获取值
template <typename T>
const typename T::ValueType* GetValueByPointer(const T& root, const GenericPointer<typename T::ValueType>& pointer, size_t* unresolvedTokenIndex = 0) {
    return pointer.Get(root, unresolvedTokenIndex);
}

// 辅助函数，用于通过通用指针获取值
template <typename T, typename CharType, size_t N>
typename T::ValueType* GetValueByPointer(T& root, const CharType (&source)[N], size_t* unresolvedTokenIndex = 0) {
    return GenericPointer<typename T::ValueType>(source, N - 1).Get(root, unresolvedTokenIndex);
}
// 通过指针获取值，如果未解析的令牌索引不为零，则返回指向值的指针
template <typename T, typename CharType, size_t N>
const typename T::ValueType* GetValueByPointer(const T& root, const CharType(&source)[N], size_t* unresolvedTokenIndex = 0) {
    return GenericPointer<typename T::ValueType>(source, N - 1).Get(root, unresolvedTokenIndex);
}

//////////////////////////////////////////////////////////////////////////////

// 通过指针获取值，如果找不到指定的值，则返回默认值
template <typename T>
typename T::ValueType& GetValueByPointerWithDefault(T& root, const GenericPointer<typename T::ValueType>& pointer, const typename T::ValueType& defaultValue, typename T::AllocatorType& a) {
    return pointer.GetWithDefault(root, defaultValue, a);
}

// 通过指针获取值，如果找不到指定的值，则返回默认值
template <typename T>
typename T::ValueType& GetValueByPointerWithDefault(T& root, const GenericPointer<typename T::ValueType>& pointer, const typename T::Ch* defaultValue, typename T::AllocatorType& a) {
    return pointer.GetWithDefault(root, defaultValue, a);
}

// 通过指针获取值，如果找不到指定的值，则返回默认值
#if RAPIDJSON_HAS_STDSTRING
template <typename T>
typename T::ValueType& GetValueByPointerWithDefault(T& root, const GenericPointer<typename T::ValueType>& pointer, const std::basic_string<typename T::Ch>& defaultValue, typename T::AllocatorType& a) {
    return pointer.GetWithDefault(root, defaultValue, a);
}
#endif

// 通过指针获取值，如果找不到指定的值，则返回默认值
template <typename T, typename T2>
RAPIDJSON_DISABLEIF_RETURN((internal::OrExpr<internal::IsPointer<T2>, internal::IsGenericValue<T2> >), (typename T::ValueType&))
GetValueByPointerWithDefault(T& root, const GenericPointer<typename T::ValueType>& pointer, T2 defaultValue, typename T::AllocatorType& a) {
    return pointer.GetWithDefault(root, defaultValue, a);
}

// 通过指针获取值，如果找不到指定的值，则返回默认值
template <typename T, typename CharType, size_t N>
typename T::ValueType& GetValueByPointerWithDefault(T& root, const CharType(&source)[N], const typename T::ValueType& defaultValue, typename T::AllocatorType& a) {
    return GenericPointer<typename T::ValueType>(source, N - 1).GetWithDefault(root, defaultValue, a);
}

// 通过指针获取值，如果找不到指定的值，则返回默认值
template <typename T, typename CharType, size_t N>
// 通过指针获取具有默认值的值，如果找不到指定的值，则返回默认值
typename T::ValueType& GetValueByPointerWithDefault(T& root, const CharType(&source)[N], const typename T::Ch* defaultValue, typename T::AllocatorType& a) {
    return GenericPointer<typename T::ValueType>(source, N - 1).GetWithDefault(root, defaultValue, a);
}

// 如果编译器支持 std::string，则通过指针获取具有默认值的值，如果找不到指定的值，则返回默认值
template <typename T, typename CharType, size_t N>
typename T::ValueType& GetValueByPointerWithDefault(T& root, const CharType(&source)[N], const std::basic_string<typename T::Ch>& defaultValue, typename T::AllocatorType& a) {
    return GenericPointer<typename T::ValueType>(source, N - 1).GetWithDefault(root, defaultValue, a);
}

// 如果 T2 不是指针或通用值类型，则通过指针获取具有默认值的值，如果找不到指定的值，则返回默认值
template <typename T, typename CharType, size_t N, typename T2>
RAPIDJSON_DISABLEIF_RETURN((internal::OrExpr<internal::IsPointer<T2>, internal::IsGenericValue<T2> >), (typename T::ValueType&))
GetValueByPointerWithDefault(T& root, const CharType(&source)[N], T2 defaultValue, typename T::AllocatorType& a) {
    return GenericPointer<typename T::ValueType>(source, N - 1).GetWithDefault(root, defaultValue, a);
}

// 没有分配器参数，通过指针获取具有默认值的值，如果找不到指定的值，则返回默认值
template <typename DocumentType>
typename DocumentType::ValueType& GetValueByPointerWithDefault(DocumentType& document, const GenericPointer<typename DocumentType::ValueType>& pointer, const typename DocumentType::ValueType& defaultValue) {
    return pointer.GetWithDefault(document, defaultValue);
}

// 没有分配器参数，通过指针获取具有默认值的值，如果找不到指定的值，则返回默认值
template <typename DocumentType>
typename DocumentType::ValueType& GetValueByPointerWithDefault(DocumentType& document, const GenericPointer<typename DocumentType::ValueType>& pointer, const typename DocumentType::Ch* defaultValue) {
    return pointer.GetWithDefault(document, defaultValue);
}

// 如果编译器支持 std::string，则没有分配器参数，通过指针获取具有默认值的值，如果找不到指定的值，则返回默认值
template <typename DocumentType>
typename DocumentType::ValueType& GetValueByPointerWithDefault(DocumentType& document, const GenericPointer<typename DocumentType::ValueType>& pointer, const std::basic_string<typename DocumentType::Ch>& defaultValue) {
    # 返回指针所指向的document对象的值，如果不存在则返回默认值
    return pointer.GetWithDefault(document, defaultValue);
#endif

// 如果 T2 不是指针类型或者 GenericValue 类型，则返回 DocumentType::ValueType 的引用
template <typename DocumentType, typename T2>
RAPIDJSON_DISABLEIF_RETURN((internal::OrExpr<internal::IsPointer<T2>, internal::IsGenericValue<T2> >), (typename DocumentType::ValueType&))
GetValueByPointerWithDefault(DocumentType& document, const GenericPointer<typename DocumentType::ValueType>& pointer, T2 defaultValue) {
    return pointer.GetWithDefault(document, defaultValue);
}

// 通过指针获取默认值，如果找不到则返回默认值
template <typename DocumentType, typename CharType, size_t N>
typename DocumentType::ValueType& GetValueByPointerWithDefault(DocumentType& document, const CharType(&source)[N], const typename DocumentType::ValueType& defaultValue) {
    return GenericPointer<typename DocumentType::ValueType>(source, N - 1).GetWithDefault(document, defaultValue);
}

// 通过指针获取默认值，如果找不到则返回默认值
template <typename DocumentType, typename CharType, size_t N>
typename DocumentType::ValueType& GetValueByPointerWithDefault(DocumentType& document, const CharType(&source)[N], const typename DocumentType::Ch* defaultValue) {
    return GenericPointer<typename DocumentType::ValueType>(source, N - 1).GetWithDefault(document, defaultValue);
}

#if RAPIDJSON_HAS_STDSTRING
// 如果支持 std::basic_string，则通过指针获取默认值，如果找不到则返回默认值
template <typename DocumentType, typename CharType, size_t N>
typename DocumentType::ValueType& GetValueByPointerWithDefault(DocumentType& document, const CharType(&source)[N], const std::basic_string<typename DocumentType::Ch>& defaultValue) {
    return GenericPointer<typename DocumentType::ValueType>(source, N - 1).GetWithDefault(document, defaultValue);
}
#endif

// 如果 T2 不是指针类型或者 GenericValue 类型，则返回 DocumentType::ValueType 的引用
template <typename DocumentType, typename CharType, size_t N, typename T2>
RAPIDJSON_DISABLEIF_RETURN((internal::OrExpr<internal::IsPointer<T2>, internal::IsGenericValue<T2> >), (typename DocumentType::ValueType&))
GetValueByPointerWithDefault(DocumentType& document, const CharType(&source)[N], T2 defaultValue) {
    return GenericPointer<typename DocumentType::ValueType>(source, N - 1).GetWithDefault(document, defaultValue);
}
// 定义模板函数，根据指针设置值
template <typename T>
typename T::ValueType& SetValueByPointer(T& root, const GenericPointer<typename T::ValueType>& pointer, typename T::ValueType& value, typename T::AllocatorType& a) {
    return pointer.Set(root, value, a);
}

// 定义模板函数，根据指针设置值
template <typename T>
typename T::ValueType& SetValueByPointer(T& root, const GenericPointer<typename T::ValueType>& pointer, const typename T::ValueType& value, typename T::AllocatorType& a) {
    return pointer.Set(root, value, a);
}

// 定义模板函数，根据指针设置值
template <typename T>
typename T::ValueType& SetValueByPointer(T& root, const GenericPointer<typename T::ValueType>& pointer, const typename T::Ch* value, typename T::AllocatorType& a) {
    return pointer.Set(root, value, a);
}

// 如果支持 std::basic_string，则定义模板函数，根据指针设置值
#if RAPIDJSON_HAS_STDSTRING
template <typename T>
typename T::ValueType& SetValueByPointer(T& root, const GenericPointer<typename T::ValueType>& pointer, const std::basic_string<typename T::Ch>& value, typename T::AllocatorType& a) {
    return pointer.Set(root, value, a);
}
#endif

// 定义模板函数，根据指针设置值，排除指针和通用值类型
template <typename T, typename T2>
RAPIDJSON_DISABLEIF_RETURN((internal::OrExpr<internal::IsPointer<T2>, internal::IsGenericValue<T2> >), (typename T::ValueType&))
SetValueByPointer(T& root, const GenericPointer<typename T::ValueType>& pointer, T2 value, typename T::AllocatorType& a) {
    return pointer.Set(root, value, a);
}

// 定义模板函数，根据指针设置值
template <typename T, typename CharType, size_t N>
typename T::ValueType& SetValueByPointer(T& root, const CharType(&source)[N], typename T::ValueType& value, typename T::AllocatorType& a) {
    return GenericPointer<typename T::ValueType>(source, N - 1).Set(root, value, a);
}

// 定义模板函数，根据指针设置值
template <typename T, typename CharType, size_t N>
typename T::ValueType& SetValueByPointer(T& root, const CharType(&source)[N], const typename T::ValueType& value, typename T::AllocatorType& a) {
    return GenericPointer<typename T::ValueType>(source, N - 1).Set(root, value, a);
}
// 通过指针设置值的模板函数，接受目标对象、源字符串、值和分配器作为参数，返回设置后的值
template <typename T, typename CharType, size_t N>
typename T::ValueType& SetValueByPointer(T& root, const CharType(&source)[N], const typename T::Ch* value, typename T::AllocatorType& a) {
    return GenericPointer<typename T::ValueType>(source, N - 1).Set(root, value, a);
}

// 如果编译器支持 std::basic_string，则重载模板函数，接受目标对象、源字符串、std::basic_string 值和分配器作为参数，返回设置后的值
#if RAPIDJSON_HAS_STDSTRING
template <typename T, typename CharType, size_t N>
typename T::ValueType& SetValueByPointer(T& root, const CharType(&source)[N], const std::basic_string<typename T::Ch>& value, typename T::AllocatorType& a) {
    return GenericPointer<typename T::ValueType>(source, N - 1).Set(root, value, a);
}
#endif

// 如果 T2 不是指针或通用值类型，则重载模板函数，接受目标对象、源字符串、值和分配器作为参数，返回设置后的值
template <typename T, typename CharType, size_t N, typename T2>
RAPIDJSON_DISABLEIF_RETURN((internal::OrExpr<internal::IsPointer<T2>, internal::IsGenericValue<T2> >), (typename T::ValueType&))
SetValueByPointer(T& root, const CharType(&source)[N], T2 value, typename T::AllocatorType& a) {
    return GenericPointer<typename T::ValueType>(source, N - 1).Set(root, value, a);
}

// 不带分配器参数的模板函数，接受文档类型对象、指针和值作为参数，返回设置后的值
template <typename DocumentType>
typename DocumentType::ValueType& SetValueByPointer(DocumentType& document, const GenericPointer<typename DocumentType::ValueType>& pointer, typename DocumentType::ValueType& value) {
    return pointer.Set(document, value);
}

// 不带分配器参数的模板函数，接受文档类型对象、指针和常量值作为参数，返回设置后的值
template <typename DocumentType>
typename DocumentType::ValueType& SetValueByPointer(DocumentType& document, const GenericPointer<typename DocumentType::ValueType>& pointer, const typename DocumentType::ValueType& value) {
    return pointer.Set(document, value);
}

// 不带分配器参数的模板函数，接受文档类型对象、指针和常量字符串值作为参数，返回设置后的值
template <typename DocumentType>
typename DocumentType::ValueType& SetValueByPointer(DocumentType& document, const GenericPointer<typename DocumentType::ValueType>& pointer, const typename DocumentType::Ch* value) {
    return pointer.Set(document, value);
}

// 如果编译器支持 std::basic_string，则重载模板函数
#if RAPIDJSON_HAS_STDSTRING
template <typename DocumentType>
// 为给定指针设置数值，返回设置后的数值引用
typename DocumentType::ValueType& SetValueByPointer(DocumentType& document, const GenericPointer<typename DocumentType::ValueType>& pointer, const std::basic_string<typename DocumentType::Ch>& value) {
    return pointer.Set(document, value);
}
#endif

// 如果 T2 不是指针或者通用数值类型，则返回 DocumentType::ValueType 的引用
template <typename DocumentType, typename T2>
RAPIDJSON_DISABLEIF_RETURN((internal::OrExpr<internal::IsPointer<T2>, internal::IsGenericValue<T2> >), (typename DocumentType::ValueType&))
SetValueByPointer(DocumentType& document, const GenericPointer<typename DocumentType::ValueType>& pointer, T2 value) {
    return pointer.Set(document, value);
}

// 为给定指针设置数值，返回设置后的数值引用
template <typename DocumentType, typename CharType, size_t N>
typename DocumentType::ValueType& SetValueByPointer(DocumentType& document, const CharType(&source)[N], typename DocumentType::ValueType& value) {
    return GenericPointer<typename DocumentType::ValueType>(source, N - 1).Set(document, value);
}

// 为给定指针设置数值，返回设置后的数值引用
template <typename DocumentType, typename CharType, size_t N>
typename DocumentType::ValueType& SetValueByPointer(DocumentType& document, const CharType(&source)[N], const typename DocumentType::ValueType& value) {
    return GenericPointer<typename DocumentType::ValueType>(source, N - 1).Set(document, value);
}

// 为给定指针设置数值，返回设置后的数值引用
template <typename DocumentType, typename CharType, size_t N>
typename DocumentType::ValueType& SetValueByPointer(DocumentType& document, const CharType(&source)[N], const typename DocumentType::Ch* value) {
    return GenericPointer<typename DocumentType::ValueType>(source, N - 1).Set(document, value);
}

#if RAPIDJSON_HAS_STDSTRING
// 为给定指针设置数值，返回设置后的数值引用
template <typename DocumentType, typename CharType, size_t N>
typename DocumentType::ValueType& SetValueByPointer(DocumentType& document, const CharType(&source)[N], const std::basic_string<typename DocumentType::Ch>& value) {
    return GenericPointer<typename DocumentType::ValueType>(source, N - 1).Set(document, value);
}
#endif

// 为给定指针设置数值，返回设置后的数值引用
template <typename DocumentType, typename CharType, size_t N, typename T2>
// 使用 DisableIf 返回类型为 typename DocumentType::ValueType& 的函数，根据指针设置文档中的值
RAPIDJSON_DISABLEIF_RETURN((internal::OrExpr<internal::IsPointer<T2>, internal::IsGenericValue<T2> >), (typename DocumentType::ValueType&))
SetValueByPointer(DocumentType& document, const CharType(&source)[N], T2 value) {
    return GenericPointer<typename DocumentType::ValueType>(source, N - 1).Set(document, value);
}

//////////////////////////////////////////////////////////////////////////////

// 通过指针交换值的函数模板，使用指定的分配器
template <typename T>
typename T::ValueType& SwapValueByPointer(T& root, const GenericPointer<typename T::ValueType>& pointer, typename T::ValueType& value, typename T::AllocatorType& a) {
    return pointer.Swap(root, value, a);
}

// 通过指针交换值的函数模板，使用指定的分配器
template <typename T, typename CharType, size_t N>
typename T::ValueType& SwapValueByPointer(T& root, const CharType(&source)[N], typename T::ValueType& value, typename T::AllocatorType& a) {
    return GenericPointer<typename T::ValueType>(source, N - 1).Swap(root, value, a);
}

// 通过指针交换值的函数模板
template <typename DocumentType>
typename DocumentType::ValueType& SwapValueByPointer(DocumentType& document, const GenericPointer<typename DocumentType::ValueType>& pointer, typename DocumentType::ValueType& value) {
    return pointer.Swap(document, value);
}

// 通过指针交换值的函数模板
template <typename DocumentType, typename CharType, size_t N>
typename DocumentType::ValueType& SwapValueByPointer(DocumentType& document, const CharType(&source)[N], typename DocumentType::ValueType& value) {
    return GenericPointer<typename DocumentType::ValueType>(source, N - 1).Swap(document, value);
}

//////////////////////////////////////////////////////////////////////////////

// 通过指针删除值的函数模板
template <typename T>
bool EraseValueByPointer(T& root, const GenericPointer<typename T::ValueType>& pointer) {
    return pointer.Erase(root);
}

// 通过指针删除值的函数模板
template <typename T, typename CharType, size_t N>
bool EraseValueByPointer(T& root, const CharType(&source)[N]) {
    return GenericPointer<typename T::ValueType>(source, N - 1).Erase(root);
}

//@}

// 结束 RapidJSON 命名空间
RAPIDJSON_NAMESPACE_END

// 如果是 clang 或者是 MSC 编译器，则执行以下代码
#if defined(__clang__) || defined(_MSC_VER)
# 取消对诊断的忽略
RAPIDJSON_DIAG_POP
# 结束对头文件的条件编译
#endif
# 结束对头文件 RAPIDJSON_POINTER_H_ 的条件编译
#endif // RAPIDJSON_POINTER_H_
```