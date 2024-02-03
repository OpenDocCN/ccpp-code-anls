# `xmrig\src\3rdparty\rapidjson\schema.h`

```cpp
// 定义了 RAPIDJSON_SCHEMA_H_ 宏，用于避免重复包含
#ifndef RAPIDJSON_SCHEMA_H_
#define RAPIDJSON_SCHEMA_H_

// 包含必要的头文件
#include "document.h"
#include "pointer.h"
#include "stringbuffer.h"
#include "error/en.h"
#include "uri.h"
#include <cmath> // abs, floor

// 定义 RAPIDJSON_SCHEMA_USE_INTERNALREGEX 宏，用于选择是否使用内部正则表达式
#if !defined(RAPIDJSON_SCHEMA_USE_INTERNALREGEX)
#define RAPIDJSON_SCHEMA_USE_INTERNALREGEX 1
#else
#define RAPIDJSON_SCHEMA_USE_INTERNALREGEX 0
#endif

// 根据条件定义 RAPIDJSON_SCHEMA_USE_STDREGEX 宏，用于选择是否使用标准库的正则表达式
#if !RAPIDJSON_SCHEMA_USE_INTERNALREGEX && defined(RAPIDJSON_SCHEMA_USE_STDREGEX) && (__cplusplus >=201103L || (defined(_MSC_VER) && _MSC_VER >= 1800))
#define RAPIDJSON_SCHEMA_USE_STDREGEX 1
#else
#define RAPIDJSON_SCHEMA_USE_STDREGEX 0
#endif

// 根据条件定义 RAPIDJSON_SCHEMA_HAS_REGEX 宏，用于判断是否有正则表达式支持
#if RAPIDJSON_SCHEMA_USE_INTERNALREGEX || RAPIDJSON_SCHEMA_USE_STDREGEX
#define RAPIDJSON_SCHEMA_HAS_REGEX 1
#else
#define RAPIDJSON_SCHEMA_HAS_REGEX 0
#endif

// 定义 RAPIDJSON_SCHEMA_VERBOSE 宏，用于控制是否输出详细信息
#ifndef RAPIDJSON_SCHEMA_VERBOSE
#define RAPIDJSON_SCHEMA_VERBOSE 0
#endif

// 根据条件选择是否关闭特定的编译警告
RAPIDJSON_DIAG_PUSH

#if defined(__GNUC__)
RAPIDJSON_DIAG_OFF(effc++)
#endif

#ifdef __clang__
RAPIDJSON_DIAG_OFF(weak-vtables)
RAPIDJSON_DIAG_OFF(exit-time-destructors)
RAPIDJSON_DIAG_OFF(c++98-compat-pedantic)
RAPIDJSON_DIAG_OFF(variadic-macros)
#elif defined(_MSC_VER)
RAPIDJSON_DIAG_OFF(4512) // 禁止特定编译器警告
#endif

RAPIDJSON_NAMESPACE_BEGIN

///////////////////////////////////////////////////////////////////////////////
// Verbose Utilities

#if RAPIDJSON_SCHEMA_VERBOSE

namespace internal {

inline void PrintInvalidKeyword(const char* keyword) {
    printf("Fail keyword: %s\n", keyword);
}

inline void PrintInvalidKeyword(const wchar_t* keyword) {
    wprintf(L"Fail keyword: %ls\n", keyword);
}

inline void PrintInvalidDocument(const char* document) {
    printf("Fail document: %s\n\n", document);
}

inline void PrintInvalidDocument(const wchar_t* document) {
    wprintf(L"Fail document: %ls\n\n", document);
}

inline void PrintValidatorPointers(unsigned depth, const char* s, const char* d) {
    printf("S: %*s%s\nD: %*s%s\n\n", depth * 4, " ", s, depth * 4, " ", d);
}

inline void PrintValidatorPointers(unsigned depth, const wchar_t* s, const wchar_t* d) {
    wprintf(L"S: %*ls%ls\nD: %*ls%ls\n\n", depth * 4, L" ", s, depth * 4, L" ", d);
}

} // namespace internal

#endif // RAPIDJSON_SCHEMA_VERBOSE

///////////////////////////////////////////////////////////////////////////////
// RAPIDJSON_INVALID_KEYWORD_RETURN

#if RAPIDJSON_SCHEMA_VERBOSE
#define RAPIDJSON_INVALID_KEYWORD_VERBOSE(keyword) internal::PrintInvalidKeyword(keyword)
#else
#define RAPIDJSON_INVALID_KEYWORD_VERBOSE(keyword)
#endif

#define RAPIDJSON_INVALID_KEYWORD_RETURN(code)\
RAPIDJSON_MULTILINEMACRO_BEGIN\
    context.invalidCode = code;\
    context.invalidKeyword = SchemaType::GetValidateErrorKeyword(code).GetString();\
    RAPIDJSON_INVALID_KEYWORD_VERBOSE(context.invalidKeyword);\
    return false;\
RAPIDJSON_MULTILINEMACRO_END

///////////////////////////////////////////////////////////////////////////////
// ValidateFlag

/*! \def RAPIDJSON_VALIDATE_DEFAULT_FLAGS
    \ingroup RAPIDJSON_CONFIG
    \brief User-defined kValidateDefaultFlags definition.
    # 用户可以将其定义为任何 \c ValidateFlag 组合
#ifndef RAPIDJSON_VALIDATE_DEFAULT_FLAGS
#define RAPIDJSON_VALIDATE_DEFAULT_FLAGS kValidateNoFlags
#endif

// 如果未定义RAPIDJSON_VALIDATE_DEFAULT_FLAGS，则定义为kValidateNoFlags


enum ValidateFlag {
    kValidateNoFlags = 0,                                       //!< No flags are set.
    kValidateContinueOnErrorFlag = 1,                           //!< Don't stop after first validation error.
    kValidateDefaultFlags = RAPIDJSON_VALIDATE_DEFAULT_FLAGS    //!< Default validate flags. Can be customized by defining RAPIDJSON_VALIDATE_DEFAULT_FLAGS
};

// 定义枚举类型ValidateFlag，包含三个枚举值，分别表示无标志、继续在错误后继续验证、默认验证标志


template <typename ValueType, typename Allocator>
class GenericSchemaDocument;

// 声明泛型类GenericSchemaDocument，用于表示通用的模式文档


namespace internal {

template <typename SchemaDocumentType>
class Schema;

// 命名空间internal中声明模板类Schema，用于表示模式文档


class ISchemaValidator {
public:
    virtual ~ISchemaValidator() {}
    virtual bool IsValid() const = 0;
    virtual void SetValidateFlags(unsigned flags) = 0;
    virtual unsigned GetValidateFlags() const = 0;
};

// 声明接口类ISchemaValidator，包含纯虚函数用于验证模式


template <typename SchemaType>
class ISchemaStateFactory {
public:
    virtual ~ISchemaStateFactory() {}
    virtual ISchemaValidator* CreateSchemaValidator(const SchemaType&, const bool inheritContinueOnErrors) = 0;
    virtual void DestroySchemaValidator(ISchemaValidator* validator) = 0;
    virtual void* CreateHasher() = 0;
    virtual uint64_t GetHashCode(void* hasher) = 0;
    virtual void DestroryHasher(void* hasher) = 0;
    virtual void* MallocState(size_t size) = 0;
    virtual void FreeState(void* p) = 0;
};

// 声明模板类ISchemaStateFactory，包含纯虚函数用于创建和销毁模式验证器、哈希器以及分配和释放状态


template <typename SchemaType>
class IValidationErrorHandler {
public:
    typedef typename SchemaType::Ch Ch;

// 声明模板类IValidationErrorHandler，用于处理验证错误，包含类型别名Ch用于表示字符类型
    // 定义一个类型别名 SValue，其类型为 SchemaType::SValue
    typedef typename SchemaType::SValue SValue;

    // 虚析构函数，用于释放资源
    virtual ~IValidationErrorHandler() {}

    // 当实际值不是期望值的倍数时调用，传入实际值和期望值
    virtual void NotMultipleOf(int64_t actual, const SValue& expected) = 0;
    virtual void NotMultipleOf(uint64_t actual, const SValue& expected) = 0;
    virtual void NotMultipleOf(double actual, const SValue& expected) = 0;

    // 当实际值超过期望最大值时调用，传入实际值、期望值和是否为排他性的标志
    virtual void AboveMaximum(int64_t actual, const SValue& expected, bool exclusive) = 0;
    virtual void AboveMaximum(uint64_t actual, const SValue& expected, bool exclusive) = 0;
    virtual void AboveMaximum(double actual, const SValue& expected, bool exclusive) = 0;

    // 当实际值低于期望最小值时调用，传入实际值、期望值和是否为排他性的标志
    virtual void BelowMinimum(int64_t actual, const SValue& expected, bool exclusive) = 0;
    virtual void BelowMinimum(uint64_t actual, const SValue& expected, bool exclusive) = 0;
    virtual void BelowMinimum(double actual, const SValue& expected, bool exclusive) = 0;

    // 当字符串长度超过期望长度时调用，传入字符串、实际长度和期望长度
    virtual void TooLong(const Ch* str, SizeType length, SizeType expected) = 0;
    // 当字符串长度低于期望长度时调用，传入字符串、实际长度和期望长度
    virtual void TooShort(const Ch* str, SizeType length, SizeType expected) = 0;
    // 当字符串不匹配预期长度时调用，传入字符串和实际长度
    virtual void DoesNotMatch(const Ch* str, SizeType length) = 0;

    // 当索引处的项被禁止时调用，传入索引值
    virtual void DisallowedItem(SizeType index) = 0;
    // 当实际项数少于期望项数时调用，传入实际项数和期望项数
    virtual void TooFewItems(SizeType actualCount, SizeType expectedCount) = 0;
    // 当实际项数多于期望项数时调用，传入实际项数和期望项数
    virtual void TooManyItems(SizeType actualCount, SizeType expectedCount) = 0;
    // 当存在重复项时调用，传入两个重复项的索引值
    virtual void DuplicateItems(SizeType index1, SizeType index2) = 0;

    // 当实际属性数多于期望属性数时调用，传入实际属性数和期望属性数
    virtual void TooManyProperties(SizeType actualCount, SizeType expectedCount) = 0;
    // 当实际属性数少于期望属性数时调用，传入实际属性数和期望属性数
    virtual void TooFewProperties(SizeType actualCount, SizeType expectedCount) = 0;
    // 开始缺失属性的处理
    virtual void StartMissingProperties() = 0;
    // 添加缺失的属性，传入属性名
    virtual void AddMissingProperty(const SValue& name) = 0;
    // 结束缺失属性的处理，返回布尔值
    virtual bool EndMissingProperties() = 0;
    // 属性违规，传入子验证器和数量
    virtual void PropertyViolations(ISchemaValidator** subvalidators, SizeType count) = 0;
    // 当属性被禁止时调用，传入属性名和长度
    virtual void DisallowedProperty(const Ch* name, SizeType length) = 0;

    // 开始依赖错误的处理
    virtual void StartDependencyErrors() = 0;
    // 开始缺失依赖属性的处理
    virtual void StartMissingDependentProperties() = 0;
    # 添加缺失的依赖属性
    virtual void AddMissingDependentProperty(const SValue& targetName) = 0;
    
    # 结束缺失的依赖属性
    virtual void EndMissingDependentProperties(const SValue& sourceName) = 0;
    
    # 添加依赖模式错误
    virtual void AddDependencySchemaError(const SValue& souceName, ISchemaValidator* subvalidator) = 0;
    
    # 结束依赖错误
    virtual bool EndDependencyErrors() = 0;
    
    # 不允许的值
    virtual void DisallowedValue(const ValidateErrorCode code) = 0;
    
    # 开始不允许的类型
    virtual void StartDisallowedType() = 0;
    
    # 添加期望的类型
    virtual void AddExpectedType(const typename SchemaType::ValueType& expectedType) = 0;
    
    # 结束不允许的类型
    virtual void EndDisallowedType(const typename SchemaType::ValueType& actualType) = 0;
    
    # 不是所有的
    virtual void NotAllOf(ISchemaValidator** subvalidators, SizeType count) = 0;
    
    # 一个都不是
    virtual void NoneOf(ISchemaValidator** subvalidators, SizeType count) = 0;
    
    # 不是其中之一
    virtual void NotOneOf(ISchemaValidator** subvalidators, SizeType count, bool matched) = 0;
    
    # 不允许
    virtual void Disallowed() = 0;
};

///////////////////////////////////////////////////////////////////////////////
// Hasher

// 用于比较复合值的哈希器
template<typename Encoding, typename Allocator>
class Hasher {
public:
    typedef typename Encoding::Ch Ch;

    // 构造函数，初始化栈
    Hasher(Allocator* allocator = 0, size_t stackCapacity = kDefaultSize) : stack_(allocator, stackCapacity) {}

    // 空值处理
    bool Null() { return WriteType(kNullType); }
    // 布尔值处理
    bool Bool(bool b) { return WriteType(b ? kTrueType : kFalseType); }
    // 整数处理
    bool Int(int i) { Number n; n.u.i = i; n.d = static_cast<double>(i); return WriteNumber(n); }
    // 无符号整数处理
    bool Uint(unsigned u) { Number n; n.u.u = u; n.d = static_cast<double>(u); return WriteNumber(n); }
    // 64位整数处理
    bool Int64(int64_t i) { Number n; n.u.i = i; n.d = static_cast<double>(i); return WriteNumber(n); }
    // 64位无符号整数处理
    bool Uint64(uint64_t u) { Number n; n.u.u = u; n.d = static_cast<double>(u); return WriteNumber(n); }
    // 双精度浮点数处理
    bool Double(double d) {
        Number n;
        if (d < 0) n.u.i = static_cast<int64_t>(d);
        else       n.u.u = static_cast<uint64_t>(d);
        n.d = d;
        return WriteNumber(n);
    }

    // 原始数字处理
    bool RawNumber(const Ch* str, SizeType len, bool) {
        WriteBuffer(kNumberType, str, len * sizeof(Ch));
        return true;
    }

    // 字符串处理
    bool String(const Ch* str, SizeType len, bool) {
        WriteBuffer(kStringType, str, len * sizeof(Ch));
        return true;
    }

    // 开始对象处理
    bool StartObject() { return true; }
    // 键处理
    bool Key(const Ch* str, SizeType len, bool copy) { return String(str, len, copy); }
    // 结束对象处理
    bool EndObject(SizeType memberCount) {
        uint64_t h = Hash(0, kObjectType);
        uint64_t* kv = stack_.template Pop<uint64_t>(memberCount * 2);
        for (SizeType i = 0; i < memberCount; i++)
            h ^= Hash(kv[i * 2], kv[i * 2 + 1]);  // 使用异或实现成员顺序无关
        *stack_.template Push<uint64_t>() = h;
        return true;
    }

    // 开始数组处理
    bool StartArray() { return true; }
    # 结束数组的解析，并对数组元素进行哈希计算
    bool EndArray(SizeType elementCount) {
        # 计算数组类型的哈希值
        uint64_t h = Hash(0, kArrayType);
        # 从栈中弹出指定数量的元素
        uint64_t* e = stack_.template Pop<uint64_t>(elementCount);
        # 遍历弹出的元素，使用哈希值计算元素的顺序敏感哈希值
        for (SizeType i = 0; i < elementCount; i++)
            h = Hash(h, e[i]); // Use hash to achieve element order sensitive
        # 将计算得到的哈希值压入栈中
        *stack_.template Push<uint64_t>() = h;
        # 返回解析结果为真
        return true;
    }
    
    # 检查栈中元素是否有效
    bool IsValid() const { return stack_.GetSize() == sizeof(uint64_t); }
    
    # 获取栈顶元素的哈希值
    uint64_t GetHashCode() const {
        # 断言栈中元素有效
        RAPIDJSON_ASSERT(IsValid());
        # 返回栈顶元素的哈希值
        return *stack_.template Top<uint64_t>();
    }
private:
    // 默认大小为256
    static const size_t kDefaultSize = 256;
    // 定义一个Number结构体
    struct Number {
        // 定义一个联合体U，包含一个无符号整数和一个有符号整数
        union U {
            uint64_t u;
            int64_t i;
        }u;
        // 一个双精度浮点数
        double d;
    };

    // 写入类型信息
    bool WriteType(Type type) { return WriteBuffer(type, 0, 0); }

    // 写入Number类型数据
    bool WriteNumber(const Number& n) { return WriteBuffer(kNumberType, &n, sizeof(n)); }

    // 写入缓冲区
    bool WriteBuffer(Type type, const void* data, size_t len) {
        // 使用FNV-1a哈希算法计算哈希值
        uint64_t h = Hash(RAPIDJSON_UINT64_C2(0x84222325, 0xcbf29ce4), type);
        const unsigned char* d = static_cast<const unsigned char*>(data);
        for (size_t i = 0; i < len; i++)
            h = Hash(h, d[i]);
        // 将哈希值压入栈中
        *stack_.template Push<uint64_t>() = h;
        return true;
    }

    // 计算哈希值
    static uint64_t Hash(uint64_t h, uint64_t d) {
        static const uint64_t kPrime = RAPIDJSON_UINT64_C2(0x00000100, 0x000001b3);
        h ^= d;
        h *= kPrime;
        return h;
    }

    // 分配器的栈
    Stack<Allocator> stack_;
};

///////////////////////////////////////////////////////////////////////////////
// SchemaValidationContext

template <typename SchemaDocumentType>
struct SchemaValidationContext {
    typedef Schema<SchemaDocumentType> SchemaType;
    typedef ISchemaStateFactory<SchemaType> SchemaValidatorFactoryType;
    typedef IValidationErrorHandler<SchemaType> ErrorHandlerType;
    typedef typename SchemaType::ValueType ValueType;
    typedef typename ValueType::Ch Ch;

    // 模式验证器类型枚举
    enum PatternValidatorType {
        kPatternValidatorOnly,
        kPatternValidatorWithProperty,
        kPatternValidatorWithAdditionalProperty
    };
    # 构造函数，初始化 SchemaValidationContext 对象
    SchemaValidationContext(SchemaValidatorFactoryType& f, ErrorHandlerType& eh, const SchemaType* s) :
        factory(f),  # 初始化 factory 属性
        error_handler(eh),  # 初始化 error_handler 属性
        schema(s),  # 初始化 schema 属性
        valueSchema(),  # 初始化 valueSchema 属性
        invalidKeyword(),  # 初始化 invalidKeyword 属性
        invalidCode(),  # 初始化 invalidCode 属性
        hasher(),  # 初始化 hasher 属性
        arrayElementHashCodes(),  # 初始化 arrayElementHashCodes 属性
        validators(),  # 初始化 validators 属性
        validatorCount(),  # 初始化 validatorCount 属性
        patternPropertiesValidators(),  # 初始化 patternPropertiesValidators 属性
        patternPropertiesValidatorCount(),  # 初始化 patternPropertiesValidatorCount 属性
        patternPropertiesSchemas(),  # 初始化 patternPropertiesSchemas 属性
        patternPropertiesSchemaCount(),  # 初始化 patternPropertiesSchemaCount 属性
        valuePatternValidatorType(kPatternValidatorOnly),  # 初始化 valuePatternValidatorType 属性
        propertyExist(),  # 初始化 propertyExist 属性
        inArray(false),  # 初始化 inArray 属性
        valueUniqueness(false),  # 初始化 valueUniqueness 属性
        arrayUniqueness(false)  # 初始化 arrayUniqueness 属性
    {
    }

    # 析构函数，释放 SchemaValidationContext 对象占用的资源
    ~SchemaValidationContext() {
        if (hasher)  # 如果 hasher 存在
            factory.DestroryHasher(hasher);  # 调用 factory 的销毁哈希器方法
        if (validators) {  # 如果 validators 存在
            for (SizeType i = 0; i < validatorCount; i++)  # 遍历 validators
                factory.DestroySchemaValidator(validators[i]);  # 调用 factory 的销毁模式验证器方法
            factory.FreeState(validators);  # 释放 validators 占用的内存
        }
        if (patternPropertiesValidators) {  # 如果 patternPropertiesValidators 存在
            for (SizeType i = 0; i < patternPropertiesValidatorCount; i++)  # 遍历 patternPropertiesValidators
                factory.DestroySchemaValidator(patternPropertiesValidators[i]);  # 调用 factory 的销毁模式验证器方法
            factory.FreeState(patternPropertiesValidators);  # 释放 patternPropertiesValidators 占用的内存
        }
        if (patternPropertiesSchemas)  # 如果 patternPropertiesSchemas 存在
            factory.FreeState(patternPropertiesSchemas);  # 释放 patternPropertiesSchemas 占用的内存
        if (propertyExist)  # 如果 propertyExist 存在
            factory.FreeState(propertyExist);  # 释放 propertyExist 占用的内存
    }

    SchemaValidatorFactoryType& factory;  # 工厂对象的引用
    ErrorHandlerType& error_handler;  # 错误处理器的引用
    const SchemaType* schema;  # 模式类型的指针
    const SchemaType* valueSchema;  # 值模式类型的指针
    const Ch* invalidKeyword;  # 无效关键字的指针
    ValidateErrorCode invalidCode;  # 无效错误码
    void* hasher;  # 哈希器指针，只有验证器可以访问
    void* arrayElementHashCodes;  # 数组元素哈希码指针，只有验证器可以访问
    ISchemaValidator** validators;  # 模式验证器指针数组
    SizeType validatorCount;  # 验证器数量
    ISchemaValidator** patternPropertiesValidators;  # 模式属性验证器指针数组
    SizeType patternPropertiesValidatorCount;  # 模式属性验证器数量
    const SchemaType** patternPropertiesSchemas;  # 模式属性类型指针数组
    SizeType patternPropertiesSchemaCount;  # 模式属性类型数量
    # 定义变量，用于存储值的模式验证器类型
    PatternValidatorType valuePatternValidatorType;
    # 定义变量，用于存储对象的模式验证器类型
    PatternValidatorType objectPatternValidatorType;
    # 定义变量，用于存储数组元素的索引
    SizeType arrayElementIndex;
    # 定义指针变量，用于标记属性是否存在
    bool* propertyExist;
    # 定义变量，用于标记当前是否在数组中
    bool inArray;
    # 定义变量，用于标记值的唯一性
    bool valueUniqueness;
    # 定义变量，用于标记数组的唯一性
    bool arrayUniqueness;
// 结束类定义
};

///////////////////////////////////////////////////////////////////////////////
// Schema

// 模板类 Schema，使用 SchemaDocumentType 作为模板参数
template <typename SchemaDocumentType>
class Schema {
public:
    // 定义类型别名
    typedef typename SchemaDocumentType::ValueType ValueType;
    typedef typename SchemaDocumentType::AllocatorType AllocatorType;
    typedef typename SchemaDocumentType::PointerType PointerType;
    typedef typename ValueType::EncodingType EncodingType;
    typedef typename EncodingType::Ch Ch;
    typedef SchemaValidationContext<SchemaDocumentType> Context;
    typedef Schema<SchemaDocumentType> SchemaType;
    typedef GenericValue<EncodingType, AllocatorType> SValue;
    typedef IValidationErrorHandler<Schema> ErrorHandler;
    typedef GenericUri<ValueType, AllocatorType> UriType;
    // 声明 GenericSchemaDocument 类为友元类
    friend class GenericSchemaDocument<ValueType, AllocatorType>;
    # 构造函数，初始化 Schema 对象
    Schema(SchemaDocumentType* schemaDocument, const PointerType& p, const ValueType& value, const ValueType& document, AllocatorType* allocator, const UriType& id = UriType()) :
        allocator_(allocator),  # 初始化 allocator_ 成员变量
        uri_(schemaDocument->GetURI(), *allocator),  # 初始化 uri_ 成员变量
        id_(id),  # 初始化 id_ 成员变量
        pointer_(p, allocator),  # 初始化 pointer_ 成员变量
        typeless_(schemaDocument->GetTypeless()),  # 初始化 typeless_ 成员变量
        enum_(),  # 初始化 enum_ 成员变量
        enumCount_(),  # 初始化 enumCount_ 成员变量
        not_(),  # 初始化 not_ 成员变量
        type_((1 << kTotalSchemaType) - 1),  # 初始化 type_ 成员变量
        validatorCount_(),  # 初始化 validatorCount_ 成员变量
        notValidatorIndex_(),  # 初始化 notValidatorIndex_ 成员变量
        properties_(),  # 初始化 properties_ 成员变量
        additionalPropertiesSchema_(),  # 初始化 additionalPropertiesSchema_ 成员变量
        patternProperties_(),  # 初始化 patternProperties_ 成员变量
        patternPropertyCount_(),  # 初始化 patternPropertyCount_ 成员变量
        propertyCount_(),  # 初始化 propertyCount_ 成员变量
        minProperties_(),  # 初始化 minProperties_ 成员变量
        maxProperties_(SizeType(~0)),  # 初始化 maxProperties_ 成员变量
        additionalProperties_(true),  # 初始化 additionalProperties_ 成员变量
        hasDependencies_(),  # 初始化 hasDependencies_ 成员变量
        hasRequired_(),  # 初始化 hasRequired_ 成员变量
        hasSchemaDependencies_(),  # 初始化 hasSchemaDependencies_ 成员变量
        additionalItemsSchema_(),  # 初始化 additionalItemsSchema_ 成员变量
        itemsList_(),  # 初始化 itemsList_ 成员变量
        itemsTuple_(),  # 初始化 itemsTuple_ 成员变量
        itemsTupleCount_(),  # 初始化 itemsTupleCount_ 成员变量
        minItems_(),  # 初始化 minItems_ 成员变量
        maxItems_(SizeType(~0)),  # 初始化 maxItems_ 成员变量
        additionalItems_(true),  # 初始化 additionalItems_ 成员变量
        uniqueItems_(false),  # 初始化 uniqueItems_ 成员变量
        pattern_(),  # 初始化 pattern_ 成员变量
        minLength_(0),  # 初始化 minLength_ 成员变量
        maxLength_(~SizeType(0)),  # 初始化 maxLength_ 成员变量
        exclusiveMinimum_(false),  # 初始化 exclusiveMinimum_ 成员变量
        exclusiveMaximum_(false),  # 初始化 exclusiveMaximum_ 成员变量
        defaultValueLength_(0)  # 初始化 defaultValueLength_ 成员变量
    }

    # 析构函数，释放 Schema 对象占用的内存
    ~Schema() {
        AllocatorType::Free(enum_);  # 释放 enum_ 成员变量占用的内存
        if (properties_) {  # 判断 properties_ 成员变量是否为空
            for (SizeType i = 0; i < propertyCount_; i++)  # 遍历 properties_ 成员变量
                properties_[i].~Property();  # 释放 properties_ 数组中每个元素占用的内存
            AllocatorType::Free(properties_);  # 释放 properties_ 数组占用的内存
        }
        if (patternProperties_) {  # 判断 patternProperties_ 成员变量是否为空
            for (SizeType i = 0; i < patternPropertyCount_; i++)  # 遍历 patternProperties_ 成员变量
                patternProperties_[i].~PatternProperty();  # 释放 patternProperties_ 数组中每个元素占用的内存
            AllocatorType::Free(patternProperties_);  # 释放 patternProperties_ 数组占用的内存
        }
        AllocatorType::Free(itemsTuple_);  # 释放 itemsTuple_ 成员变量占用的内存
#if RAPIDJSON_SCHEMA_HAS_REGEX
        // 如果定义了RAPIDJSON_SCHEMA_HAS_REGEX，则执行以下代码块
        if (pattern_) {
            // 如果pattern_存在，则调用其析构函数
            pattern_->~RegexType();
            // 释放pattern_所占用的内存
            AllocatorType::Free(pattern_);
        }
#endif
    }

    // 返回uri_成员变量的引用
    const SValue& GetURI() const {
        return uri_;
    }

    // 返回id_成员变量的引用
    const UriType& GetId() const {
        return id_;
    }

    // 返回pointer_成员变量的引用
    const PointerType& GetPointer() const {
        return pointer_;
    }

    // 在给定的上下文中开始验证值
    bool BeginValue(Context& context) const {
        // 如果上下文中处于数组中
        if (context.inArray) {
            // 如果设置了uniqueItems_，则将valueUniqueness设置为true
            if (uniqueItems_)
                context.valueUniqueness = true;

            // 如果设置了itemsList_，则将valueSchema设置为itemsList_
            if (itemsList_)
                context.valueSchema = itemsList_;
            // 如果设置了itemsTuple_
            else if (itemsTuple_) {
                // 如果数组元素索引小于itemsTupleCount_，则将valueSchema设置为itemsTuple_中对应索引的值
                if (context.arrayElementIndex < itemsTupleCount_)
                    context.valueSchema = itemsTuple_[context.arrayElementIndex];
                // 如果设置了additionalItemsSchema_，则将valueSchema设置为additionalItemsSchema_
                else if (additionalItemsSchema_)
                    context.valueSchema = additionalItemsSchema_;
                // 如果设置了additionalItems_
                else if (additionalItems_)
                    context.valueSchema = typeless_;
                // 否则，报告不允许的数组元素，并设置valueSchema为typeless_
                else {
                    context.error_handler.DisallowedItem(context.arrayElementIndex);
                    context.valueSchema = typeless_;
                    context.arrayElementIndex++;
                    RAPIDJSON_INVALID_KEYWORD_RETURN(kValidateErrorAdditionalItems);
                }
            }
            // 否则，将valueSchema设置为typeless_
            else
                context.valueSchema = typeless_;

            // 增加数组元素索引
            context.arrayElementIndex++;
        }
        return true;
    }

    // 如果类型不包含kNullSchemaType，则报告不允许的类型，并返回错误
    bool Null(Context& context) const {
        if (!(type_ & (1 << kNullSchemaType))) {
            DisallowedType(context, GetNullString());
            RAPIDJSON_INVALID_KEYWORD_RETURN(kValidateErrorType);
        }
        // 创建并返回并行验证器
        return CreateParallelValidator(context);
    }
    # 检查是否为布尔类型，如果不是则报错
    bool Bool(Context& context, bool) const {
        if (!(type_ & (1 << kBooleanSchemaType))) {
            DisallowedType(context, GetBooleanString());
            RAPIDJSON_INVALID_KEYWORD_RETURN(kValidateErrorType);
        }
        return CreateParallelValidator(context);
    }

    # 检查是否为整数类型，如果不是则返回false
    bool Int(Context& context, int i) const {
        if (!CheckInt(context, i))
            return false;
        return CreateParallelValidator(context);
    }

    # 检查是否为无符号整数类型，如果不是则返回false
    bool Uint(Context& context, unsigned u) const {
        if (!CheckUint(context, u))
            return false;
        return CreateParallelValidator(context);
    }

    # 检查是否为64位整数类型，如果不是则返回false
    bool Int64(Context& context, int64_t i) const {
        if (!CheckInt(context, i))
            return false;
        return CreateParallelValidator(context);
    }

    # 检查是否为64位无符号整数类型，如果不是则返回false
    bool Uint64(Context& context, uint64_t u) const {
        if (!CheckUint(context, u))
            return false;
        return CreateParallelValidator(context);
    }

    # 检查是否为双精度浮点数类型，如果不是则报错
    bool Double(Context& context, double d) const {
        if (!(type_ & (1 << kNumberSchemaType))) {
            DisallowedType(context, GetNumberString());
            RAPIDJSON_INVALID_KEYWORD_RETURN(kValidateErrorType);
        }

        # 检查是否存在最小值限制，如果存在则检查是否满足最小值限制
        if (!minimum_.IsNull() && !CheckDoubleMinimum(context, d))
            return false;

        # 检查是否存在最大值限制，如果存在则检查是否满足最大值限制
        if (!maximum_.IsNull() && !CheckDoubleMaximum(context, d))
            return false;

        # 检查是否存在倍数限制，如果存在则检查是否满足倍数限制
        if (!multipleOf_.IsNull() && !CheckDoubleMultipleOf(context, d))
            return false;

        return CreateParallelValidator(context);
    }
    // 检查当前 Schema 类型是否为字符串类型，如果不是则报错
    if (!(type_ & (1 << kStringSchemaType))) {
        DisallowedType(context, GetStringString());
        RAPIDJSON_INVALID_KEYWORD_RETURN(kValidateErrorType);
    }

    // 检查字符串长度是否在规定范围内，如果不在范围内则报错
    if (minLength_ != 0 || maxLength_ != SizeType(~0)) {
        SizeType count;
        // 计算字符串的字符数
        if (internal::CountStringCodePoint<EncodingType>(str, length, &count)) {
            // 如果字符数小于最小长度，则报错
            if (count < minLength_) {
                context.error_handler.TooShort(str, length, minLength_);
                RAPIDJSON_INVALID_KEYWORD_RETURN(kValidateErrorMinLength);
            }
            // 如果字符数大于最大长度，则报错
            if (count > maxLength_) {
                context.error_handler.TooLong(str, length, maxLength_);
                RAPIDJSON_INVALID_KEYWORD_RETURN(kValidateErrorMaxLength);
            }
        }
    }

    // 检查字符串是否符合指定的模式，如果不符合则报错
    if (pattern_ && !IsPatternMatch(pattern_, str, length)) {
        context.error_handler.DoesNotMatch(str, length);
        RAPIDJSON_INVALID_KEYWORD_RETURN(kValidateErrorPattern);
    }

    // 创建并返回并行验证器
    return CreateParallelValidator(context);
    // 如果当前类型不是对象类型，则在上下文中标记为不允许的类型，并返回验证错误类型
    bool StartObject(Context& context) const {
        if (!(type_ & (1 << kObjectSchemaType))) {
            DisallowedType(context, GetObjectString());
            RAPIDJSON_INVALID_KEYWORD_RETURN(kValidateErrorType);
        }

        // 如果存在依赖项或必需项，则在上下文中分配并初始化属性存在的布尔数组
        if (hasDependencies_ || hasRequired_) {
            context.propertyExist = static_cast<bool*>(context.factory.MallocState(sizeof(bool) * propertyCount_));
            std::memset(context.propertyExist, 0, sizeof(bool) * propertyCount_);
        }

        // 如果存在模式属性，则预先分配模式数组的空间
        if (patternProperties_) {
            SizeType count = patternPropertyCount_ + 1; // 为 valuePatternValidatorType 额外分配空间
            context.patternPropertiesSchemas = static_cast<const SchemaType**>(context.factory.MallocState(sizeof(const SchemaType*) * count));
            context.patternPropertiesSchemaCount = 0;
            std::memset(context.patternPropertiesSchemas, 0, sizeof(SchemaType*) * count);
        }

        // 创建并返回并行验证器
        return CreateParallelValidator(context);
    }

    // 标记当前在数组中，并验证当前类型是否为数组类型
    bool StartArray(Context& context) const {
        context.arrayElementIndex = 0;
        context.inArray = true;  // 确保标记当前在数组中

        if (!(type_ & (1 << kArraySchemaType))) {
            DisallowedType(context, GetArrayString());
            RAPIDJSON_INVALID_KEYWORD_RETURN(kValidateErrorType);
        }

        // 创建并返回并行验证器
        return CreateParallelValidator(context);
    }

    // 结束数组验证，并检查元素数量是否符合最小和最大要求
    bool EndArray(Context& context, SizeType elementCount) const {
        context.inArray = false;

        if (elementCount < minItems_) {
            context.error_handler.TooFewItems(elementCount, minItems_);
            RAPIDJSON_INVALID_KEYWORD_RETURN(kValidateErrorMinItems);
        }

        if (elementCount > maxItems_) {
            context.error_handler.TooManyItems(elementCount, maxItems_);
            RAPIDJSON_INVALID_KEYWORD_RETURN(kValidateErrorMaxItems);
        }

        return true;
    }

    // 根据 Ch 生成字符串文字的函数
# 定义一个宏，用于生成静态字符串值
#define RAPIDJSON_STRING_(name, ...) \
    # 生成一个静态函数，返回指定字符串值的 ValueType 对象
    static const ValueType& Get##name##String() {\
        # 生成一个静态字符数组，包含传入的字符串值和结束符'\0'
        static const Ch s[] = { __VA_ARGS__, '\0' };\
        # 生成一个静态的 ValueType 对象，使用字符数组和长度初始化
        static const ValueType v(s, static_cast<SizeType>(sizeof(s) / sizeof(Ch) - 1));\
        # 返回该静态对象
        return v;\
    }
    # 调用宏，生成对应的静态字符串值函数
    RAPIDJSON_STRING_(Null, 'n', 'u', 'l', 'l')
    RAPIDJSON_STRING_(Boolean, 'b', 'o', 'o', 'l', 'e', 'a', 'n')
    RAPIDJSON_STRING_(Object, 'o', 'b', 'j', 'e', 'c', 't')
    RAPIDJSON_STRING_(Array, 'a', 'r', 'r', 'a', 'y')
    RAPIDJSON_STRING_(String, 's', 't', 'r', 'i', 'n', 'g')
    RAPIDJSON_STRING_(Number, 'n', 'u', 'm', 'b', 'e', 'r')
    RAPIDJSON_STRING_(Integer, 'i', 'n', 't', 'e', 'g', 'e', 'r')
    RAPIDJSON_STRING_(Type, 't', 'y', 'p', 'e')
    RAPIDJSON_STRING_(Enum, 'e', 'n', 'u', 'm')
    RAPIDJSON_STRING_(AllOf, 'a', 'l', 'l', 'O', 'f')
    RAPIDJSON_STRING_(AnyOf, 'a', 'n', 'y', 'O', 'f')
    RAPIDJSON_STRING_(OneOf, 'o', 'n', 'e', 'O', 'f')
    RAPIDJSON_STRING_(Not, 'n', 'o', 't')
    RAPIDJSON_STRING_(Properties, 'p', 'r', 'o', 'p', 'e', 'r', 't', 'i', 'e', 's')
    RAPIDJSON_STRING_(Required, 'r', 'e', 'q', 'u', 'i', 'r', 'e', 'd')
    RAPIDJSON_STRING_(Dependencies, 'd', 'e', 'p', 'e', 'n', 'd', 'e', 'n', 'c', 'i', 'e', 's')
    RAPIDJSON_STRING_(PatternProperties, 'p', 'a', 't', 't', 'e', 'r', 'n', 'P', 'r', 'o', 'p', 'e', 'r', 't', 'i', 'e', 's')
    RAPIDJSON_STRING_(AdditionalProperties, 'a', 'd', 'd', 'i', 't', 'i', 'o', 'n', 'a', 'l', 'P', 'r', 'o', 'p', 'e', 'r', 't', 'i', 'e', 's')
    RAPIDJSON_STRING_(MinProperties, 'm', 'i', 'n', 'P', 'r', 'o', 'p', 'e', 'r', 't', 'i', 'e', 's')
    RAPIDJSON_STRING_(MaxProperties, 'm', 'a', 'x', 'P', 'r', 'o', 'p', 'e', 'r', 't', 'i', 'e', 's')
    RAPIDJSON_STRING_(Items, 'i', 't', 'e', 'm', 's')
    RAPIDJSON_STRING_(MinItems, 'm', 'i', 'n', 'I', 't', 'e', 'm', 's')
    RAPIDJSON_STRING_(MaxItems, 'm', 'a', 'x', 'I', 't', 'e', 'm', 's')
    # 定义字符串常量 AdditionalItems
    RAPIDJSON_STRING_(AdditionalItems, 'a', 'd', 'd', 'i', 't', 'i', 'o', 'n', 'a', 'l', 'I', 't', 'e', 'm', 's')
    # 定义字符串常量 UniqueItems
    RAPIDJSON_STRING_(UniqueItems, 'u', 'n', 'i', 'q', 'u', 'e', 'I', 't', 'e', 'm', 's')
    # 定义字符串常量 MinLength
    RAPIDJSON_STRING_(MinLength, 'm', 'i', 'n', 'L', 'e', 'n', 'g', 't', 'h')
    # 定义字符串常量 MaxLength
    RAPIDJSON_STRING_(MaxLength, 'm', 'a', 'x', 'L', 'e', 'n', 'g', 't', 'h')
    # 定义字符串常量 Pattern
    RAPIDJSON_STRING_(Pattern, 'p', 'a', 't', 't', 'e', 'r', 'n')
    # 定义字符串常量 Minimum
    RAPIDJSON_STRING_(Minimum, 'm', 'i', 'n', 'i', 'm', 'u', 'm')
    # 定义字符串常量 Maximum
    RAPIDJSON_STRING_(Maximum, 'm', 'a', 'x', 'i', 'm', 'u', 'm')
    # 定义字符串常量 ExclusiveMinimum
    RAPIDJSON_STRING_(ExclusiveMinimum, 'e', 'x', 'c', 'l', 'u', 's', 'i', 'v', 'e', 'M', 'i', 'n', 'i', 'm', 'u', 'm')
    # 定义字符串常量 ExclusiveMaximum
    RAPIDJSON_STRING_(ExclusiveMaximum, 'e', 'x', 'c', 'l', 'u', 's', 'i', 'v', 'e', 'M', 'a', 'x', 'i', 'm', 'u', 'm')
    # 定义字符串常量 MultipleOf
    RAPIDJSON_STRING_(MultipleOf, 'm', 'u', 'l', 't', 'i', 'p', 'l', 'e', 'O', 'f')
    # 定义字符串常量 DefaultValue
    RAPIDJSON_STRING_(DefaultValue, 'd', 'e', 'f', 'a', 'u', 'l', 't')
    # 定义字符串常量 Ref
    RAPIDJSON_STRING_(Ref, '$', 'r', 'e', 'f')
    # 定义字符串常量 Id
    RAPIDJSON_STRING_(Id, 'i', 'd')
    
    # 定义字符串常量 SchemeEnd
    RAPIDJSON_STRING_(SchemeEnd, ':')
    # 定义字符串常量 AuthStart
    RAPIDJSON_STRING_(AuthStart, '/', '/')
    # 定义字符串常量 QueryStart
    RAPIDJSON_STRING_(QueryStart, '?')
    # 定义字符串常量 FragStart
    RAPIDJSON_STRING_(FragStart, '#')
    # 定义字符串常量 Slash
    RAPIDJSON_STRING_(Slash, '/')
    # 定义字符串常量 Dot
    RAPIDJSON_STRING_(Dot, '.')
// 取消定义 RAPIDJSON_STRING_
#undef RAPIDJSON_STRING_

private:
    // 定义枚举类型 SchemaValueType，表示不同的模式类型
    enum SchemaValueType {
        kNullSchemaType,
        kBooleanSchemaType,
        kObjectSchemaType,
        kArraySchemaType,
        kStringSchemaType,
        kNumberSchemaType,
        kIntegerSchemaType,
        kTotalSchemaType
    };

    // 根据宏定义选择不同的正则表达式类型
#if RAPIDJSON_SCHEMA_USE_INTERNALREGEX
        typedef internal::GenericRegex<EncodingType, AllocatorType> RegexType;
#elif RAPIDJSON_SCHEMA_USE_STDREGEX
        typedef std::basic_regex<Ch> RegexType;
#else
        typedef char RegexType;
#endif

    // 定义结构体 SchemaArray，表示模式数组
    struct SchemaArray {
        SchemaArray() : schemas(), count() {}
        ~SchemaArray() { AllocatorType::Free(schemas); }
        const SchemaType** schemas;
        SizeType begin; // 上下文验证器的起始索引
        SizeType count; // 数量
    };

    // 添加唯一元素到数组中
    template <typename V1, typename V2>
    void AddUniqueElement(V1& a, const V2& v) {
        for (typename V1::ConstValueIterator itr = a.Begin(); itr != a.End(); ++itr)
            if (*itr == v)
                return;
        V1 c(v, *allocator_);
        a.PushBack(c, *allocator_);
    }

    // 获取对象中指定成员的值
    static const ValueType* GetMember(const ValueType& value, const ValueType& name) {
        typename ValueType::ConstMemberIterator itr = value.FindMember(name);
        return itr != value.MemberEnd() ? &(itr->value) : 0;
    }

    // 如果指定成员存在且为布尔类型，则赋值给 out
    static void AssignIfExist(bool& out, const ValueType& value, const ValueType& name) {
        if (const ValueType* v = GetMember(value, name))
            if (v->IsBool())
                out = v->GetBool();
    }

    // 如果指定成员存在且为无符号整数类型，则赋值给 out
    static void AssignIfExist(SizeType& out, const ValueType& value, const ValueType& name) {
        if (const ValueType* v = GetMember(value, name))
            if (v->IsUint64() && v->GetUint64() <= SizeType(~0))
                out = static_cast<SizeType>(v->GetUint64());
    }
    // 如果值中存在指定名称的成员，则进行赋值操作
    void AssignIfExist(SchemaArray& out, SchemaDocumentType& schemaDocument, const PointerType& p, const ValueType& value, const ValueType& name, const ValueType& document) {
        // 获取指定名称的成员值
        if (const ValueType* v = GetMember(value, name)) {
            // 如果该成员是数组且长度大于0
            if (v->IsArray() && v->Size() > 0) {
                // 在指定位置添加一个新的成员，并分配内存
                PointerType q = p.Append(name, allocator_);
                // 设置输出数组的长度
                out.count = v->Size();
                // 分配内存给输出数组的schemas成员
                out.schemas = static_cast<const Schema**>(allocator_->Malloc(out.count * sizeof(const Schema*)));
                // 将输出数组的schemas成员初始化为0
                memset(out.schemas, 0, sizeof(Schema*)* out.count);
                // 遍历数组，为每个元素创建schema
                for (SizeType i = 0; i < out.count; i++)
                    schemaDocument.CreateSchema(&out.schemas[i], q.Append(i, allocator_), (*v)[i], document, id_);
                // 设置输出数组的起始位置
                out.begin = validatorCount_;
                // 更新validatorCount_
                validatorCount_ += out.count;
            }
        }
    }
#if RAPIDJSON_SCHEMA_USE_INTERNALREGEX
    // 如果使用内部正则表达式引擎，则创建模式
    template <typename ValueType>
    RegexType* CreatePattern(const ValueType& value) {
        // 如果值是字符串类型
        if (value.IsString()) {
            // 用分配器分配内存创建正则表达式对象
            RegexType* r = new (allocator_->Malloc(sizeof(RegexType))) RegexType(value.GetString(), allocator_);
            // 如果正则表达式无效
            if (!r->IsValid()) {
                // 销毁正则表达式对象
                r->~RegexType();
                // 释放内存
                AllocatorType::Free(r);
                r = 0;
            }
            return r;
        }
        return 0;
    }

    // 检查模式是否匹配
    static bool IsPatternMatch(const RegexType* pattern, const Ch *str, SizeType) {
        // 使用内部正则表达式引擎进行匹配
        GenericRegexSearch<RegexType> rs(*pattern);
        return rs.Search(str);
    }
#elif RAPIDJSON_SCHEMA_USE_STDREGEX
    // 如果使用标准库的正则表达式引擎，则创建模式
    template <typename ValueType>
    RegexType* CreatePattern(const ValueType& value) {
        // 如果值是字符串类型
        if (value.IsString()) {
            // 分配内存创建正则表达式对象
            RegexType *r = static_cast<RegexType*>(allocator_->Malloc(sizeof(RegexType)));
            try {
                // 使用标准库的正则表达式引擎创建正则表达式对象
                return new (r) RegexType(value.GetString(), std::size_t(value.GetStringLength()), std::regex_constants::ECMAScript);
            }
            // 捕获异常
            catch (const std::regex_error&) {
                // 释放内存
                AllocatorType::Free(r);
            }
        }
        return 0;
    }

    // 检查模式是否匹配
    static bool IsPatternMatch(const RegexType* pattern, const Ch *str, SizeType length) {
        // 使用标准库的正则表达式引擎进行匹配
        std::match_results<const Ch*> r;
        return std::regex_search(str, str + length, r, *pattern);
    }
#else
    // 如果不使用任何正则表达式引擎，则返回空指针
    template <typename ValueType>
    RegexType* CreatePattern(const ValueType&) { return 0; }

    // 检查模式是否匹配
    static bool IsPatternMatch(const RegexType*, const Ch *, SizeType) { return true; }
#endif // RAPIDJSON_SCHEMA_USE_STDREGEX
    // 向类型标志位中添加指定类型的标志位
    void AddType(const ValueType& type) {
        // 如果类型为 null，则将对应的标志位置为1
        if      (type == GetNullString()   ) type_ |= 1 << kNullSchemaType;
        // 如果类型为 boolean，则将对应的标志位置为1
        else if (type == GetBooleanString()) type_ |= 1 << kBooleanSchemaType;
        // 如果类型为 object，则将对应的标志位置为1
        else if (type == GetObjectString() ) type_ |= 1 << kObjectSchemaType;
        // 如果类型为 array，则将对应的标志位置为1
        else if (type == GetArrayString()  ) type_ |= 1 << kArraySchemaType;
        // 如果类型为 string，则将对应的标志位置为1
        else if (type == GetStringString() ) type_ |= 1 << kStringSchemaType;
        // 如果类型为 integer，则将对应的标志位置为1
        else if (type == GetIntegerString()) type_ |= 1 << kIntegerSchemaType;
        // 如果类型为 number，则将对应的标志位置为1
        else if (type == GetNumberString() ) type_ |= (1 << kNumberSchemaType) | (1 << kIntegerSchemaType);
    }

    // 创建并行验证器
    bool CreateParallelValidator(Context& context) const {
        // 如果存在枚举或者数组元素唯一性，则创建哈希器
        if (enum_ || context.arrayUniqueness)
            context.hasher = context.factory.CreateHasher();

        // 如果存在验证器数量
        if (validatorCount_) {
            // 断言验证器为空
            RAPIDJSON_ASSERT(context.validators == 0);
            // 分配验证器数组的内存空间
            context.validators = static_cast<ISchemaValidator**>(context.factory.MallocState(sizeof(ISchemaValidator*) * validatorCount_));
            context.validatorCount = validatorCount_;

            // 对于这些子验证器，总是在第一个失败后返回
            if (allOf_.schemas)
                CreateSchemaValidators(context, allOf_, false);

            if (anyOf_.schemas)
                CreateSchemaValidators(context, anyOf_, false);

            if (oneOf_.schemas)
                CreateSchemaValidators(context, oneOf_, false);

            // 如果存在 not_ 验证器，则创建对应的验证器
            if (not_)
                context.validators[notValidatorIndex_] = context.factory.CreateSchemaValidator(*not_, false);

            // 如果存在模式依赖
            if (hasSchemaDependencies_) {
                // 遍历属性，如果存在依赖模式，则创建对应的验证器
                for (SizeType i = 0; i < propertyCount_; i++)
                    if (properties_[i].dependenciesSchema)
                        context.validators[properties_[i].dependenciesValidatorIndex] = context.factory.CreateSchemaValidator(*properties_[i].dependenciesSchema, false);
            }
        }

        return true;
    }
    // 创建模式验证器
    void CreateSchemaValidators(Context& context, const SchemaArray& schemas, const bool inheritContinueOnErrors) const {
        // 遍历模式数组
        for (SizeType i = 0; i < schemas.count; i++)
            // 为上下文中的验证器数组赋值，使用工厂创建模式验证器
            context.validators[schemas.begin + i] = context.factory.CreateSchemaValidator(*schemas.schemas[i], inheritContinueOnErrors);
    }
    
    // 查找属性索引，时间复杂度为 O(n)
    bool FindPropertyIndex(const ValueType& name, SizeType* outIndex) const {
        // 获取属性名的长度和字符串指针
        SizeType len = name.GetStringLength();
        const Ch* str = name.GetString();
        // 遍历属性数组
        for (SizeType index = 0; index < propertyCount_; index++)
            // 检查属性名是否匹配
            if (properties_[index].name.GetStringLength() == len &&
                (std::memcmp(properties_[index].name.GetString(), str, sizeof(Ch) * len) == 0))
            {
                // 如果匹配，则将索引赋值给 outIndex，并返回 true
                *outIndex = index;
                return true;
            }
        // 如果未找到匹配的属性名，则返回 false
        return false;
    }
    
    // 检查双精度浮点数的最小值
    bool CheckDoubleMinimum(Context& context, double d) const {
        // 检查是否小于等于最小值
        if (exclusiveMinimum_ ? d <= minimum_.GetDouble() : d < minimum_.GetDouble()) {
            // 如果小于等于最小值，则调用错误处理程序，并返回相应的错误类型
            context.error_handler.BelowMinimum(d, minimum_, exclusiveMinimum_);
            RAPIDJSON_INVALID_KEYWORD_RETURN(exclusiveMinimum_ ? kValidateErrorExclusiveMinimum : kValidateErrorMinimum);
        }
        // 如果通过验证，则返回 true
        return true;
    }
    
    // 检查双精度浮点数的最大值
    bool CheckDoubleMaximum(Context& context, double d) const {
        // 检查是否大于等于最大值
        if (exclusiveMaximum_ ? d >= maximum_.GetDouble() : d > maximum_.GetDouble()) {
            // 如果大于等于最大值，则调用错误处理程序，并返回相应的错误类型
            context.error_handler.AboveMaximum(d, maximum_, exclusiveMaximum_);
            RAPIDJSON_INVALID_KEYWORD_RETURN(exclusiveMaximum_ ? kValidateErrorExclusiveMaximum : kValidateErrorMaximum);
        }
        // 如果通过验证，则返回 true
        return true;
    }
    # 检查给定的双精度浮点数是否是指定倍数
    bool CheckDoubleMultipleOf(Context& context, double d) const {
        # 获取给定浮点数的绝对值
        double a = std::abs(d), b = std::abs(multipleOf_.GetDouble());
        # 计算商
        double q = std::floor(a / b);
        # 计算余数
        double r = a - q * b;
        # 如果余数大于0，则表示不是指定倍数，触发错误处理并返回错误码
        if (r > 0.0) {
            context.error_handler.NotMultipleOf(d, multipleOf_);
            RAPIDJSON_INVALID_KEYWORD_RETURN(kValidateErrorMultipleOf);
        }
        # 返回 true
        return true;
    }

    # 处理不允许的数据类型
    void DisallowedType(Context& context, const ValueType& actualType) const {
        # 获取错误处理器
        ErrorHandler& eh = context.error_handler;
        # 开始记录不允许的数据类型
        eh.StartDisallowedType();

        # 根据位掩码判断是否允许 null 类型，并记录到错误处理器中
        if (type_ & (1 << kNullSchemaType)) eh.AddExpectedType(GetNullString());
        # 根据位掩码判断是否允许 boolean 类型，并记录到错误处理器中
        if (type_ & (1 << kBooleanSchemaType)) eh.AddExpectedType(GetBooleanString());
        # 根据位掩码判断是否允许 object 类型，并记录到错误处理器中
        if (type_ & (1 << kObjectSchemaType)) eh.AddExpectedType(GetObjectString());
        # 根据位掩码判断是否允许 array 类型，并记录到错误处理器中
        if (type_ & (1 << kArraySchemaType)) eh.AddExpectedType(GetArrayString());
        # 根据位掩码判断是否允许 string 类型，并记录到错误处理器中
        if (type_ & (1 << kStringSchemaType)) eh.AddExpectedType(GetStringString());

        # 根据位掩码判断是否允许 number 类型，并记录到错误处理器中
        if (type_ & (1 << kNumberSchemaType)) eh.AddExpectedType(GetNumberString());
        # 如果不允许 number 类型，则判断是否允许 integer 类型，并记录到错误处理器中
        else if (type_ & (1 << kIntegerSchemaType)) eh.AddExpectedType(GetIntegerString());

        # 结束记录不允许的数据类型，并传入实际数据类型
        eh.EndDisallowedType(actualType);
    }

    # 属性结构体
    struct Property {
        # 构造函数初始化成员变量
        Property() : schema(), dependenciesSchema(), dependenciesValidatorIndex(), dependencies(), required(false) {}
        # 析构函数释放内存
        ~Property() { AllocatorType::Free(dependencies); }
        # 属性名
        SValue name;
        # 属性对应的模式
        const SchemaType* schema;
        # 依赖的模式
        const SchemaType* dependenciesSchema;
        # 依赖的验证器索引
        SizeType dependenciesValidatorIndex;
        # 依赖的属性
        bool* dependencies;
        # 是否必需
        bool required;
    };

    # 模式属性结构体
    struct PatternProperty {
        # 构造函数初始化成员变量
        PatternProperty() : schema(), pattern() {}
        # 析构函数释放内存
        ~PatternProperty() {
            if (pattern) {
                pattern->~RegexType();
                AllocatorType::Free(pattern);
            }
        }
        # 模式对应的模式
        const SchemaType* schema;
        # 正则表达式模式
        RegexType* pattern;
    };

    # 分配器指针
    AllocatorType* allocator_;
    # URI 值
    SValue uri_;
    # 声明一个变量 id_，表示 URI 类型
    UriType id_;
    # 声明一个变量 pointer_，表示指针类型
    PointerType pointer_;
    # 声明一个指向常量 SchemaType 类型的指针 typeless_
    const SchemaType* typeless_;
    # 声明一个指向 uint64_t 类型的枚举数组 enum_
    uint64_t* enum_;
    # 声明一个变量 enumCount_，表示枚举数组的大小
    SizeType enumCount_;
    # 声明一个 SchemaArray 类型的数组 allOf_
    SchemaArray allOf_;
    # 声明一个 SchemaArray 类型的数组 anyOf_
    SchemaArray anyOf_;
    # 声明一个 SchemaArray 类型的数组 oneOf_
    SchemaArray oneOf_;
    # 声明一个指向常量 SchemaType 类型的指针 not_
    const SchemaType* not_;
    # 声明一个无符号整数类型的变量 type_，表示 kSchemaType 的位掩码
    unsigned type_; // bitmask of kSchemaType
    # 声明一个变量 validatorCount_，表示验证器的数量
    SizeType validatorCount_;
    # 声明一个变量 notValidatorIndex_，表示非验证器的索引

    # 声明一个指向 Property 类型的指针 properties_
    Property* properties_;
    # 声明一个指向常量 SchemaType 类型的指针 additionalPropertiesSchema_
    const SchemaType* additionalPropertiesSchema_;
    # 声明一个指向 PatternProperty 类型的指针 patternProperties_
    PatternProperty* patternProperties_;
    # 声明一个变量 patternPropertyCount_，表示模式属性的数量
    SizeType patternPropertyCount_;
    # 声明一个变量 propertyCount_，表示属性的数量
    SizeType propertyCount_;
    # 声明一个变量 minProperties_，表示最小属性数量
    SizeType minProperties_;
    # 声明一个变量 maxProperties_，表示最大属性数量
    SizeType maxProperties_;
    # 声明一个布尔变量 additionalProperties_，表示是否有额外的属性
    bool additionalProperties_;
    # 声明一个布尔变量 hasDependencies_，表示是否有依赖关系
    bool hasDependencies_;
    # 声明一个布尔变量 hasRequired_，表示是否有必需属性
    bool hasRequired_;
    # 声明一个布尔变量 hasSchemaDependencies_，表示是否有模式依赖关系

    # 声明一个指向常量 SchemaType 类型的指针 additionalItemsSchema_
    const SchemaType* additionalItemsSchema_;
    # 声明一个指向常量 SchemaType 类型的指针 itemsList_
    const SchemaType* itemsList_;
    # 声明一个指向常量 SchemaType 类型的指针数组 itemsTuple_
    const SchemaType** itemsTuple_;
    # 声明一个变量 itemsTupleCount_，表示元组项的数量
    SizeType itemsTupleCount_;
    # 声明一个变量 minItems_，表示最小项数量
    SizeType minItems_;
    # 声明一个变量 maxItems_，表示最大项数量
    SizeType maxItems_;
    # 声明一个布尔变量 additionalItems_，表示是否有额外的项
    bool additionalItems_;
    # 声明一个布尔变量 uniqueItems_，表示是否有唯一的项

    # 声明一个指向 RegexType 类型的指针 pattern_
    RegexType* pattern_;
    # 声明一个变量 minLength_，表示最小长度
    SizeType minLength_;
    # 声明一个变量 maxLength_，表示最大长度

    # 声明一个 SValue 类型的变量 minimum_
    SValue minimum_;
    # 声明一个 SValue 类型的变量 maximum_
    SValue maximum_;
    # 声明一个 SValue 类型的变量 multipleOf_
    SValue multipleOf_;
    # 声明一个布尔变量 exclusiveMinimum_，表示是否有排他的最小值
    bool exclusiveMinimum_;
    # 声明一个布尔变量 exclusiveMaximum_，表示是否有排他的最大值

    # 声明一个变量 defaultValueLength_，表示默认值的长度
    SizeType defaultValueLength_;
// 结构模板 TokenHelper 用于帮助处理索引标记
template<typename Stack, typename Ch>
struct TokenHelper {
    // 添加索引标记到文档堆栈
    RAPIDJSON_FORCEINLINE static void AppendIndexToken(Stack& documentStack, SizeType index) {
        // 在文档堆栈中添加斜杠字符
        *documentStack.template Push<Ch>() = '/';
        // 创建一个字符缓冲区
        char buffer[21];
        // 将索引转换为字符串，并计算字符串长度
        size_t length = static_cast<size_t>((sizeof(SizeType) == 4 ? u32toa(index, buffer) : u64toa(index, buffer)) - buffer);
        // 将索引转换后的字符串逐个添加到文档堆栈中
        for (size_t i = 0; i < length; i++)
            *documentStack.template Push<Ch>() = static_cast<Ch>(buffer[i]);
    }
};

// 针对 char 类型的部分特化版本，用于避免缓冲区复制
template <typename Stack>
struct TokenHelper<Stack, char> {
    // 添加索引标记到文档堆栈
    RAPIDJSON_FORCEINLINE static void AppendIndexToken(Stack& documentStack, SizeType index) {
        // 如果 SizeType 类型为 4 字节
        if (sizeof(SizeType) == 4) {
            // 在文档堆栈中添加长度为 1+10 的 char 类型缓冲区，用于存储斜杠字符和索引
            char *buffer = documentStack.template Push<char>(1 + 10); // '/' + uint
            // 添加斜杠字符
            *buffer++ = '/';
            // 将索引转换为字符串，并返回字符串结束位置
            const char* end = internal::u32toa(index, buffer);
            // 从文档堆栈中弹出多余的字符
            documentStack.template Pop<char>(static_cast<size_t>(10 - (end - buffer)));
        }
        // 如果 SizeType 类型为 8 字节
        else {
            // 在文档堆栈中添加长度为 1+20 的 char 类型缓冲区，用于存储斜杠字符和索引
            char *buffer = documentStack.template Push<char>(1 + 20); // '/' + uint64
            // 添加斜杠字符
            *buffer++ = '/';
            // 将索引转换为字符串，并返回字符串结束位置
            const char* end = internal::u64toa(index, buffer);
            // 从文档堆栈中弹出多余的字符
            documentStack.template Pop<char>(static_cast<size_t>(20 - (end - buffer)));
        }
    }
};

} // namespace internal

///////////////////////////////////////////////////////////////////////////////
// IGenericRemoteSchemaDocumentProvider

// 通用远程模式文档提供者模板类
template <typename SchemaDocumentType>
class IGenericRemoteSchemaDocumentProvider {
public:
    // 定义别名
    typedef typename SchemaDocumentType::Ch Ch;
    typedef typename SchemaDocumentType::ValueType ValueType;
    typedef typename SchemaDocumentType::AllocatorType AllocatorType;

    // 虚析构函数
    virtual ~IGenericRemoteSchemaDocumentProvider() {}
    // 获取远程文档的抽象接口
    virtual const SchemaDocumentType* GetRemoteDocument(const Ch* uri, SizeType length) = 0;
    # 获取远程文档的虚拟方法，参数为通用URI对象
    virtual const SchemaDocumentType* GetRemoteDocument(GenericUri<ValueType, AllocatorType> uri) { return GetRemoteDocument(uri.GetBaseString(), uri.GetBaseStringLength()); }
};

///////////////////////////////////////////////////////////////////////////////
// GenericSchemaDocument

//! JSON schema document.
/*!
    A JSON schema document is a compiled version of a JSON schema.
    It is basically a tree of internal::Schema.

    \note This is an immutable class (i.e. its instance cannot be modified after construction).
    \tparam ValueT Type of JSON value (e.g. \c Value ), which also determine the encoding.
    \tparam Allocator Allocator type for allocating memory of this document.
*/
template <typename ValueT, typename Allocator = CrtAllocator>
class GenericSchemaDocument {
public:
    typedef ValueT ValueType; // Define an alias for the type of JSON value
    typedef IGenericRemoteSchemaDocumentProvider<GenericSchemaDocument> IRemoteSchemaDocumentProviderType; // Define an alias for the remote schema document provider type
    typedef Allocator AllocatorType; // Define an alias for the allocator type
    typedef typename ValueType::EncodingType EncodingType; // Define an alias for the encoding type
    typedef typename EncodingType::Ch Ch; // Define an alias for the character type
    typedef internal::Schema<GenericSchemaDocument> SchemaType; // Define an alias for the schema type
    typedef GenericPointer<ValueType, Allocator> PointerType; // Define an alias for the pointer type
    typedef GenericValue<EncodingType, AllocatorType> SValue; // Define an alias for the generic value type
    typedef GenericUri<ValueType, Allocator> UriType; // Define an alias for the URI type
    friend class internal::Schema<GenericSchemaDocument>; // Allow internal::Schema to access private members
    template <typename, typename, typename>
    friend class GenericSchemaValidator; // Allow GenericSchemaValidator to access private members

    //! Constructor.
    /*!
        Compile a JSON document into schema document.

        \param document A JSON document as source.
        \param uri The base URI of this schema document for purposes of violation reporting.
        \param uriLength Length of \c name, in code points.
        \param remoteProvider An optional remote schema document provider for resolving remote reference. Can be null.
        \param allocator An optional allocator instance for allocating memory. Can be null.
        \param pointer An optional JSON pointer to the start of the schema document
    */
    // 构造函数，用于创建 GenericSchemaDocument 对象
    explicit GenericSchemaDocument(const ValueType& document, const Ch* uri = 0, SizeType uriLength = 0,
        IRemoteSchemaDocumentProviderType* remoteProvider = 0, Allocator* allocator = 0,
        const PointerType& pointer = PointerType()) :  // PR #1393
        remoteProvider_(remoteProvider),  // 设置远程提供者
        allocator_(allocator),  // 设置分配器
        ownAllocator_(),  // 初始化自有分配器
        root_(),  // 初始化根节点
        typeless_(),  // 初始化类型
        schemaMap_(allocator, kInitialSchemaMapSize),  // 初始化模式映射
        schemaRef_(allocator, kInitialSchemaRefSize)  // 初始化模式引用
    {
        if (!allocator_)  // 如果分配器为空
            ownAllocator_ = allocator_ = RAPIDJSON_NEW(Allocator)();  // 分配新的分配器

        Ch noUri[1] = {0};  // 创建空的 URI
        uri_.SetString(uri ? uri : noUri, uriLength, *allocator_);  // 设置 URI
        docId_ = UriType(uri_, allocator_);  // 设置文档 ID

        typeless_ = static_cast<SchemaType*>(allocator_->Malloc(sizeof(SchemaType)));  // 分配类型大小的内存
        new (typeless_) SchemaType(this, PointerType(), ValueType(kObjectType).Move(), ValueType(kObjectType).Move(), allocator_, docId_);  // 创建类型对象

        // 生成根模式，它将调用 CreateSchema() 来创建子模式，并在存在 $ref 时调用 HandleRefSchema()
        // PR #1393 如果提供了输入指针，则使用输入指针
        root_ = typeless_;  // 设置根节点为类型
        if (pointer.GetTokenCount() == 0) {  // 如果指针的令牌数为0
            CreateSchemaRecursive(&root_, pointer, document, document, docId_);  // 递归创建模式
        }
        else if (const ValueType* v = pointer.Get(document)) {  // 否则，如果指针在文档中存在
            CreateSchema(&root_, pointer, *v, document, docId_);  // 创建模式
        }

        RAPIDJSON_ASSERT(root_ != 0);  // 断言根节点不为空

        schemaRef_.ShrinkToFit(); // 收缩以释放所有引用的内存
    }
#if RAPIDJSON_HAS_CXX11_RVALUE_REFS
    //! 如果支持 C++11 的右值引用，则使用移动构造函数
    GenericSchemaDocument(GenericSchemaDocument&& rhs) RAPIDJSON_NOEXCEPT :
        remoteProvider_(rhs.remoteProvider_),
        allocator_(rhs.allocator_),
        ownAllocator_(rhs.ownAllocator_),
        root_(rhs.root_),
        typeless_(rhs.typeless_),
        schemaMap_(std::move(rhs.schemaMap_)),
        schemaRef_(std::move(rhs.schemaRef_)),
        uri_(std::move(rhs.uri_)),
        docId_(rhs.docId_)
    {
        rhs.remoteProvider_ = 0;
        rhs.allocator_ = 0;
        rhs.ownAllocator_ = 0;
        rhs.typeless_ = 0;
    }
#endif

    //! 析构函数
    ~GenericSchemaDocument() {
        // 循环直到 schemaMap_ 为空
        while (!schemaMap_.Empty())
            schemaMap_.template Pop<SchemaEntry>(1)->~SchemaEntry();

        // 如果 typeless_ 不为空，则释放其内存
        if (typeless_) {
            typeless_->~SchemaType();
            Allocator::Free(typeless_);
        }

        // 释放 ownAllocator_ 的内存
        RAPIDJSON_DELETE(ownAllocator_);
    }

    // 返回 URI
    const SValue& GetURI() const { return uri_; }

    //! 获取根模式
    const SchemaType& GetRoot() const { return *root_; }

private:
    //! 禁止复制
    GenericSchemaDocument(const GenericSchemaDocument&);
    //! 禁止赋值
    GenericSchemaDocument& operator=(const GenericSchemaDocument&);

    // PR #1393
    typedef const PointerType* SchemaRefPtr;

    // 结构体 SchemaEntry
    struct SchemaEntry {
        // 构造函数
        SchemaEntry(const PointerType& p, SchemaType* s, bool o, Allocator* allocator) : pointer(p, allocator), schema(s), owned(o) {}
        // 析构函数
        ~SchemaEntry() {
            // 如果 owned 为真，则释放 schema 的内存
            if (owned) {
                schema->~SchemaType();
                Allocator::Free(schema);
            }
        }
        PointerType pointer;
        SchemaType* schema;
        bool owned;
    };

    // PR #1393 改变
    // 递归创建 JSON Schema，根据给定的指针、值、文档和 ID
    void CreateSchemaRecursive(const SchemaType** schema, const PointerType& pointer, const ValueType& v, const ValueType& document, const UriType& id) {
        // 如果值的类型是对象
        if (v.GetType() == kObjectType) {
            // 创建新的 ID
            UriType newid = UriType(CreateSchema(schema, pointer, v, document, id), allocator_);

            // 遍历对象的成员
            for (typename ValueType::ConstMemberIterator itr = v.MemberBegin(); itr != v.MemberEnd(); ++itr)
                // 递归调用 CreateSchemaRecursive
                CreateSchemaRecursive(0, pointer.Append(itr->name, allocator_), itr->value, document, newid);
        }
        // 如果值的类型是数组
        else if (v.GetType() == kArrayType)
            // 遍历数组
            for (SizeType i = 0; i < v.Size(); i++)
                // 递归调用 CreateSchemaRecursive
                CreateSchemaRecursive(0, pointer.Append(i, allocator_), v[i], document, id);
    }

    // Changed by PR #1393
    // 创建 JSON Schema，根据给定的指针、值、文档和 ID
    const UriType& CreateSchema(const SchemaType** schema, const PointerType& pointer, const ValueType& v, const ValueType& document, const UriType& id) {
        // 断言指针有效
        RAPIDJSON_ASSERT(pointer.IsValid());
        // 如果值是对象
        if (v.IsObject()) {
            // 获取指定指针的模式
            if (const SchemaType* sc = GetSchema(pointer)) {
                // 如果 schema 不为空，将指针指向模式
                if (schema)
                    *schema = sc;
                // 添加模式引用
                AddSchemaRefs(const_cast<SchemaType*>(sc));
            }
            // 如果没有找到模式
            else if (!HandleRefSchema(pointer, schema, v, document, id)) {
                // 创建新的模式
                SchemaType* s = new (allocator_->Malloc(sizeof(SchemaType))) SchemaType(this, pointer, v, document, allocator_, id);
                // 如果 schema 不为空，将指针指向新的模式
                if (schema)
                    *schema = s;
                // 返回新模式的 ID
                return s->GetId();
            }
        }
        // 如果值不是对象
        else {
            // 如果 schema 不为空，将指针指向 typeless_
            if (schema)
                *schema = typeless_;
            // 添加 typeless_ 的模式引用
            AddSchemaRefs(typeless_);
        }
        // 返回给定的 ID
        return id;
    }

    // Changed by PR #1393
    // TODO should this return a UriType& ?
    }

    //! 查找第一个具有匹配指定 URI 的已解析 'id' 的子模式。
    // 如果完全指定使用所有 URI，否则忽略片段。
    // 如果找到，返回指向子模式和其 JSON 指针的指针。
    // TODO 缓存指针 <-> id 映射
    ValueType* FindId(const ValueType& doc, const UriType& finduri, PointerType& resptr, const UriType& baseuri, bool full, const PointerType& here = PointerType()) const {
        SizeType i = 0;
        ValueType* resval = 0;
        UriType tempuri = UriType(finduri, allocator_);
        UriType localuri = UriType(baseuri, allocator_);
        if (doc.GetType() == kObjectType) {
            // 确定此对象的基本 URI
            typename ValueType::ConstMemberIterator m = doc.FindMember(SchemaType::GetIdString());
            if (m != doc.MemberEnd() && m->value.GetType() == kStringType) {
                localuri = UriType(m->value, allocator_).Resolve(baseuri, allocator_);
            }
            // 查看是否匹配
            if (localuri.Match(finduri, full)) {
                resval = const_cast<ValueType *>(&doc);
                resptr = here;
                return resval;
            }
            // 没有匹配，继续查找
            for (m = doc.MemberBegin(); m != doc.MemberEnd(); ++m) {
                if (m->value.GetType() == kObjectType || m->value.GetType() == kArrayType) {
                    resval = FindId(m->value, finduri, resptr, localuri, full, here.Append(m->name.GetString(), m->name.GetStringLength(), allocator_));
                }
                if (resval) break;
            }
        } else if (doc.GetType() == kArrayType) {
            // 继续查找
            for (typename ValueType::ConstValueIterator v = doc.Begin(); v != doc.End(); ++v) {
                if (v->GetType() == kObjectType || v->GetType() == kArrayType) {
                    resval = FindId(*v, finduri, resptr, localuri, full, here.Append(i, allocator_));
                }
                if (resval) break;
                i++;
            }
        }
        return resval;
    }

    // 由 PR #1393 添加
    // 向给定的模式添加引用
    void AddSchemaRefs(SchemaType* schema) {
        // 当引用不为空时循环
        while (!schemaRef_.Empty()) {
            // 从引用栈中弹出一个引用
            SchemaRefPtr *ref = schemaRef_.template Pop<SchemaRefPtr>(1);
            // 在模式映射中添加一个新的模式条目
            SchemaEntry *entry = schemaMap_.template Push<SchemaEntry>();
            // 使用引用创建一个新的模式条目
            new (entry) SchemaEntry(**ref, schema, false, allocator_);
        }
    }

    // 由 PR #1393 添加
    // 检查给定指针是否存在循环引用
    bool IsCyclicRef(const PointerType& pointer) const {
        // 遍历引用栈
        for (const SchemaRefPtr* ref = schemaRef_.template Bottom<SchemaRefPtr>(); ref != schemaRef_.template End<SchemaRefPtr>(); ++ref)
            // 如果指针等于引用指针，则存在循环引用
            if (pointer == **ref)
                return true;
        // 不存在循环引用
        return false;
    }

    // 获取给定指针对应的模式
    const SchemaType* GetSchema(const PointerType& pointer) const {
        // 遍历模式映射
        for (const SchemaEntry* target = schemaMap_.template Bottom<SchemaEntry>(); target != schemaMap_.template End<SchemaEntry>(); ++target)
            // 如果指针等于目标指针，则返回对应的模式
            if (pointer == target->pointer)
                return target->schema;
        // 未找到对应的模式
        return 0;
    }

    // 获取给定模式对应的指针
    PointerType GetPointer(const SchemaType* schema) const {
        // 遍历模式映射
        for (const SchemaEntry* target = schemaMap_.template Bottom<SchemaEntry>(); target != schemaMap_.template End<SchemaEntry>(); ++target)
            // 如果模式等于目标模式，则返回对应的指针
            if (schema == target->schema)
                return target->pointer;
        // 未找到对应的指针
        return PointerType();
    }

    // 获取无类型的模式
    const SchemaType* GetTypeless() const { return typeless_; }

    // 初始模式映射大小
    static const size_t kInitialSchemaMapSize = 64;
    // 初始引用栈大小
    static const size_t kInitialSchemaRefSize = 64;

    // 远程模式文档提供程序
    IRemoteSchemaDocumentProviderType* remoteProvider_;
    // 分配器
    Allocator *allocator_;
    // 自有分配器
    Allocator *ownAllocator_;
    // 根模式
    const SchemaType* root_;                //!< Root schema.
    // 无类型的模式
    SchemaType* typeless_;
    // 存储创建的指针到模式的映射
    internal::Stack<Allocator> schemaMap_;  // Stores created Pointer -> Schemas
    // 存储从 $ref(s) 直到解析的指针
    internal::Stack<Allocator> schemaRef_;  // Stores Pointer(s) from $ref(s) until resolved
    // 模式文档 URI
    SValue uri_;
    // 文档 ID URI
    UriType docId_;
};

//! GenericSchemaDocument using Value type.
typedef GenericSchemaDocument<Value> SchemaDocument;
//! IGenericRemoteSchemaDocumentProvider using SchemaDocument.
typedef IGenericRemoteSchemaDocumentProvider<SchemaDocument> IRemoteSchemaDocumentProvider;

///////////////////////////////////////////////////////////////////////////////
// GenericSchemaValidator

//! JSON Schema Validator.
/*!
    A SAX style JSON schema validator.
    It uses a \c GenericSchemaDocument to validate SAX events.
    It delegates the incoming SAX events to an output handler.
    The default output handler does nothing.
    It can be reused multiple times by calling \c Reset().

    \tparam SchemaDocumentType Type of schema document.
    \tparam OutputHandler Type of output handler. Default handler does nothing.
    \tparam StateAllocator Allocator for storing the internal validation states.
*/
template <
    typename SchemaDocumentType,
    typename OutputHandler = BaseReaderHandler<typename SchemaDocumentType::SchemaType::EncodingType>,
    typename StateAllocator = CrtAllocator>
class GenericSchemaValidator :
    public internal::ISchemaStateFactory<typename SchemaDocumentType::SchemaType>,
    public internal::ISchemaValidator,
    public internal::IValidationErrorHandler<typename SchemaDocumentType::SchemaType> {
public:
    typedef typename SchemaDocumentType::SchemaType SchemaType;
    typedef typename SchemaDocumentType::PointerType PointerType;
    typedef typename SchemaType::EncodingType EncodingType;
    typedef typename SchemaType::SValue SValue;
    typedef typename EncodingType::Ch Ch;
    typedef GenericStringRef<Ch> StringRefType;
    typedef GenericValue<EncodingType, StateAllocator> ValueType;

    //! Constructor without output handler.
    # 创建一个通用模式验证器，用于验证给定的模式文档
    # schemaDocument: 要符合的模式文档
    # allocator: 用于存储内部验证状态的可选分配器
    # schemaStackCapacity: 模式路径堆栈的可选初始容量
    # documentStackCapacity: 文档路径堆栈的可选初始容量
    GenericSchemaValidator(
        const SchemaDocumentType& schemaDocument,
        StateAllocator* allocator = 0,
        size_t schemaStackCapacity = kDefaultSchemaStackCapacity,
        size_t documentStackCapacity = kDefaultDocumentStackCapacity)
        :
        # 设置模式文档指针
        schemaDocument_(&schemaDocument),
        # 获取模式文档的根节点
        root_(schemaDocument.GetRoot()),
        # 设置状态分配器
        stateAllocator_(allocator),
        # 拥有自己的状态分配器
        ownStateAllocator_(0),
        # 使用分配器和指定的容量创建模式路径堆栈
        schemaStack_(allocator, schemaStackCapacity),
        # 使用分配器和指定的容量创建文档路径堆栈
        documentStack_(allocator, documentStackCapacity),
        # 输出处理程序初始化为空
        outputHandler_(0),
        # 错误对象初始化为对象类型
        error_(kObjectType),
        # 当前错误初始化为空
        currentError_(),
        # 丢失的依赖项初始化为空
        missingDependents_(),
        # 验证状态初始化为真
        valid_(true),
        # 标志初始化为默认验证标志
        flags_(kValidateDefaultFlags)
#if RAPIDJSON_SCHEMA_VERBOSE
        , depth_(0)
#endif
    {
    }

这是一个条件编译指令，根据RAPIDJSON_SCHEMA_VERBOSE的定义来决定是否包含depth_的初始化。


    //! Constructor with output handler.
    /*!
        \param schemaDocument The schema document to conform to.
        \param allocator Optional allocator for storing internal validation states.
        \param schemaStackCapacity Optional initial capacity of schema path stack.
        \param documentStackCapacity Optional initial capacity of document path stack.
    */
    GenericSchemaValidator(
        const SchemaDocumentType& schemaDocument,
        OutputHandler& outputHandler,
        StateAllocator* allocator = 0,
        size_t schemaStackCapacity = kDefaultSchemaStackCapacity,
        size_t documentStackCapacity = kDefaultDocumentStackCapacity)
        :
        schemaDocument_(&schemaDocument),
        root_(schemaDocument.GetRoot()),
        stateAllocator_(allocator),
        ownStateAllocator_(0),
        schemaStack_(allocator, schemaStackCapacity),
        documentStack_(allocator, documentStackCapacity),
        outputHandler_(&outputHandler),
        error_(kObjectType),
        currentError_(),
        missingDependents_(),
        valid_(true),
        flags_(kValidateDefaultFlags)
#if RAPIDJSON_SCHEMA_VERBOSE
        , depth_(0)
#endif
    {
    }

这是GenericSchemaValidator的构造函数，用于初始化GenericSchemaValidator对象的各个成员变量。


    //! Destructor.
    ~GenericSchemaValidator() {
        Reset();
        RAPIDJSON_DELETE(ownStateAllocator_);
    }

这是GenericSchemaValidator的析构函数，用于释放资源并进行清理操作。


    //! Reset the internal states.
    void Reset() {
        while (!schemaStack_.Empty())
            PopSchema();
        documentStack_.Clear();
        ResetError();
    }

这是一个重置内部状态的函数，用于清空schemaStack_和documentStack_，并调用ResetError()函数重置错误状态。


    //! Reset the error state.
    void ResetError() {
        error_.SetObject();
        currentError_.SetNull();
        missingDependents_.SetNull();
        valid_ = true;
    }

这是一个重置错误状态的函数，用于将error_设置为一个空的JSON对象，将currentError_和missingDependents_设置为null，并将valid_设置为true。


    //! Implementation of ISchemaValidator
    void SetValidateFlags(unsigned flags) {
        flags_ = flags;
    }
    virtual unsigned GetValidateFlags() const {
        return flags_;
    }

这是ISchemaValidator接口的实现，用于设置和获取验证标志。


    //! Checks whether the current state is valid.

这是一个注释，说明下面的代码用于检查当前状态是否有效。
    // 实现 ISchemaValidator 接口的 IsValid 方法，用于检查当前对象是否有效
    virtual bool IsValid() const {
        // 如果 valid_ 为假，则返回 false
        if (!valid_) return false;
        // 如果设置了继续在错误发生时继续运行，并且 error_ 不为空，则返回 false
        if (GetContinueOnErrors() && !error_.ObjectEmpty()) return false;
        // 否则返回 true
        return true;
    }

    //! 获取错误对象
    ValueType& GetError() { return error_; }
    const ValueType& GetError() const { return error_; }

    //! 获取指向无效模式的 JSON 指针
    // 如果报告所有错误，则堆栈将为空
    PointerType GetInvalidSchemaPointer() const {
        return schemaStack_.Empty() ? PointerType() : CurrentSchema().GetPointer();
    }

    //! 获取无效模式的关键字
    // 如果报告所有错误，则堆栈将为空，因此返回 "errors"
    const Ch* GetInvalidSchemaKeyword() const {
        if (!schemaStack_.Empty()) return CurrentContext().invalidKeyword;
        if (GetContinueOnErrors() && !error_.ObjectEmpty()) return (const Ch*)GetErrorsString();
        return 0;
    }

    //! 获取无效模式的错误代码
    // 如果报告所有错误，则堆栈将为空，因此返回 kValidateErrors
    ValidateErrorCode GetInvalidSchemaCode() const {
        if (!schemaStack_.Empty()) return CurrentContext().invalidCode;
        if (GetContinueOnErrors() && !error_.ObjectEmpty()) return kValidateErrors;
        return kValidateErrorNone;
    }

    //! 获取指向无效值的 JSON 指针
    // 如果报告所有错误，则堆栈将为空
    PointerType GetInvalidDocumentPointer() const {
        if (documentStack_.Empty()) {
            return PointerType();
        }
        else {
            return PointerType(documentStack_.template Bottom<Ch>(), documentStack_.GetSize() / sizeof(Ch));
        }
    }

    // 当值不是期望的倍数时，添加错误信息
    void NotMultipleOf(int64_t actual, const SValue& expected) {
        AddNumberError(kValidateErrorMultipleOf, ValueType(actual).Move(), expected);
    }
    // 如果实际值不是期望值的倍数，则添加数字错误
    void NotMultipleOf(uint64_t actual, const SValue& expected) {
        AddNumberError(kValidateErrorMultipleOf, ValueType(actual).Move(), expected);
    }
    // 如果实际值不是期望值的倍数，则添加数字错误
    void NotMultipleOf(double actual, const SValue& expected) {
        AddNumberError(kValidateErrorMultipleOf, ValueType(actual).Move(), expected);
    }
    // 如果实际值超过期望值的最大值，则添加数字错误
    void AboveMaximum(int64_t actual, const SValue& expected, bool exclusive) {
        AddNumberError(exclusive ? kValidateErrorExclusiveMaximum : kValidateErrorMaximum, ValueType(actual).Move(), expected,
            exclusive ? &SchemaType::GetExclusiveMaximumString : 0);
    }
    // 如果实际值超过期望值的最大值，则添加数字错误
    void AboveMaximum(uint64_t actual, const SValue& expected, bool exclusive) {
        AddNumberError(exclusive ? kValidateErrorExclusiveMaximum : kValidateErrorMaximum, ValueType(actual).Move(), expected,
            exclusive ? &SchemaType::GetExclusiveMaximumString : 0);
    }
    // 如果实际值超过期望值的最大值，则添加数字错误
    void AboveMaximum(double actual, const SValue& expected, bool exclusive) {
        AddNumberError(exclusive ? kValidateErrorExclusiveMaximum : kValidateErrorMaximum, ValueType(actual).Move(), expected,
            exclusive ? &SchemaType::GetExclusiveMaximumString : 0);
    }
    // 如果实际值低于期望值的最小值，则添加数字错误
    void BelowMinimum(int64_t actual, const SValue& expected, bool exclusive) {
        AddNumberError(exclusive ? kValidateErrorExclusiveMinimum : kValidateErrorMinimum, ValueType(actual).Move(), expected,
            exclusive ? &SchemaType::GetExclusiveMinimumString : 0);
    }
    // 如果实际值低于期望值的最小值，则添加数字错误
    void BelowMinimum(uint64_t actual, const SValue& expected, bool exclusive) {
        AddNumberError(exclusive ? kValidateErrorExclusiveMinimum : kValidateErrorMinimum, ValueType(actual).Move(), expected,
            exclusive ? &SchemaType::GetExclusiveMinimumString : 0);
    }
    // 如果实际值低于期望值的最小值，则添加数字错误
    void BelowMinimum(double actual, const SValue& expected, bool exclusive) {
        AddNumberError(exclusive ? kValidateErrorExclusiveMinimum : kValidateErrorMinimum, ValueType(actual).Move(), expected,
            exclusive ? &SchemaType::GetExclusiveMinimumString : 0);
    // 当字符串长度超过预期时，添加长度错误信息
    void TooLong(const Ch* str, SizeType length, SizeType expected) {
        AddNumberError(kValidateErrorMaxLength,
            ValueType(str, length, GetStateAllocator()).Move(), SValue(expected).Move());
    }
    // 当字符串长度不足预期时，添加长度错误信息
    void TooShort(const Ch* str, SizeType length, SizeType expected) {
        AddNumberError(kValidateErrorMinLength,
            ValueType(str, length, GetStateAllocator()).Move(), SValue(expected).Move());
    }
    // 当字符串不匹配预期模式时，添加模式错误信息
    void DoesNotMatch(const Ch* str, SizeType length) {
        currentError_.SetObject();
        currentError_.AddMember(GetActualString(), ValueType(str, length, GetStateAllocator()).Move(), GetStateAllocator());
        AddCurrentError(kValidateErrorPattern);
    }
    // 当存在不允许的项时，添加不允许的项错误信息
    void DisallowedItem(SizeType index) {
        currentError_.SetObject();
        currentError_.AddMember(GetDisallowedString(), ValueType(index).Move(), GetStateAllocator());
        AddCurrentError(kValidateErrorAdditionalItems, true);
    }
    // 当项数量不足预期时，添加最小项数错误信息
    void TooFewItems(SizeType actualCount, SizeType expectedCount) {
        AddNumberError(kValidateErrorMinItems,
            ValueType(actualCount).Move(), SValue(expectedCount).Move());
    }
    // 当项数量超过预期时，添加最大项数错误信息
    void TooManyItems(SizeType actualCount, SizeType expectedCount) {
        AddNumberError(kValidateErrorMaxItems,
            ValueType(actualCount).Move(), SValue(expectedCount).Move());
    }
    // 当存在重复项时，添加重复项错误信息
    void DuplicateItems(SizeType index1, SizeType index2) {
        ValueType duplicates(kArrayType);
        duplicates.PushBack(index1, GetStateAllocator());
        duplicates.PushBack(index2, GetStateAllocator());
        currentError_.SetObject();
        currentError_.AddMember(GetDuplicatesString(), duplicates, GetStateAllocator());
        AddCurrentError(kValidateErrorUniqueItems, true);
    }
    // 当属性数量超过预期时，添加最大属性数错误信息
    void TooManyProperties(SizeType actualCount, SizeType expectedCount) {
        AddNumberError(kValidateErrorMaxProperties,
            ValueType(actualCount).Move(), SValue(expectedCount).Move());
    }
    // 当属性数量过少时，添加错误信息
    void TooFewProperties(SizeType actualCount, SizeType expectedCount) {
        AddNumberError(kValidateErrorMinProperties,
            ValueType(actualCount).Move(), SValue(expectedCount).Move());
    }
    // 开始缺少属性的错误信息
    void StartMissingProperties() {
        currentError_.SetArray();
    }
    // 添加缺少的属性信息
    void AddMissingProperty(const SValue& name) {
        currentError_.PushBack(ValueType(name, GetStateAllocator()).Move(), GetStateAllocator());
    }
    // 结束缺少属性的错误信息
    bool EndMissingProperties() {
        if (currentError_.Empty())
            return false;
        ValueType error(kObjectType);
        error.AddMember(GetMissingString(), currentError_, GetStateAllocator());
        currentError_ = error;
        AddCurrentError(kValidateErrorRequired);
        return true;
    }
    // 属性违规时的处理
    void PropertyViolations(ISchemaValidator** subvalidators, SizeType count) {
        for (SizeType i = 0; i < count; ++i)
            MergeError(static_cast<GenericSchemaValidator*>(subvalidators[i])->GetError());
    }
    // 禁止的属性处理
    void DisallowedProperty(const Ch* name, SizeType length) {
        currentError_.SetObject();
        currentError_.AddMember(GetDisallowedString(), ValueType(name, length, GetStateAllocator()).Move(), GetStateAllocator());
        AddCurrentError(kValidateErrorAdditionalProperties, true);
    }

    // 开始依赖错误信息
    void StartDependencyErrors() {
        currentError_.SetObject();
    }
    // 开始缺少依赖属性的错误信息
    void StartMissingDependentProperties() {
        missingDependents_.SetArray();
    }
    // 添加缺少的依赖属性信息
    void AddMissingDependentProperty(const SValue& targetName) {
        missingDependents_.PushBack(ValueType(targetName, GetStateAllocator()).Move(), GetStateAllocator());
    }
    // 当存在缺失的依赖属性时，添加相应的错误信息
    void EndMissingDependentProperties(const SValue& sourceName) {
        if (!missingDependents_.Empty()) {
            // 创建等效的“required”错误
            ValueType error(kObjectType);
            ValidateErrorCode code = kValidateErrorRequired;
            // 添加缺失的依赖属性到错误信息中
            error.AddMember(GetMissingString(), missingDependents_.Move(), GetStateAllocator());
            // 添加错误代码
            AddErrorCode(error, code);
            // 添加错误实例的位置信息
            AddErrorInstanceLocation(error, false);
            // 当追加到指针时，确保使用其分配器
            PointerType schemaRef = GetInvalidSchemaPointer().Append(SchemaType::GetValidateErrorKeyword(kValidateErrorDependencies), &GetInvalidSchemaPointer().GetAllocator());
            // 添加错误的模式位置信息
            AddErrorSchemaLocation(error, schemaRef.Append(sourceName.GetString(), sourceName.GetStringLength(), &GetInvalidSchemaPointer().GetAllocator()));
            ValueType wrapper(kObjectType);
            wrapper.AddMember(ValueType(SchemaType::GetValidateErrorKeyword(code), GetStateAllocator()).Move(), error, GetStateAllocator());
            currentError_.AddMember(ValueType(sourceName, GetStateAllocator()).Move(), wrapper, GetStateAllocator());
        }
    }

    // 添加依赖模式的错误信息
    void AddDependencySchemaError(const SValue& sourceName, ISchemaValidator* subvalidator) {
        currentError_.AddMember(ValueType(sourceName, GetStateAllocator()).Move(),
            static_cast<GenericSchemaValidator*>(subvalidator)->GetError(), GetStateAllocator());
    }

    // 结束依赖错误信息的处理
    bool EndDependencyErrors() {
        if (currentError_.ObjectEmpty())
            return false;
        ValueType error(kObjectType);
        error.AddMember(GetErrorsString(), currentError_, GetStateAllocator());
        currentError_ = error;
        AddCurrentError(kValidateErrorDependencies);
        return true;
    }

    // 添加不允许的值的错误信息
    void DisallowedValue(const ValidateErrorCode code = kValidateErrorEnum) {
        currentError_.SetObject();
        AddCurrentError(code);
    }
    // 开始不允许的类型验证，将当前错误设置为数组
    void StartDisallowedType() {
        currentError_.SetArray();
    }
    
    // 添加期望的类型到当前错误数组中
    void AddExpectedType(const typename SchemaType::ValueType& expectedType) {
        currentError_.PushBack(ValueType(expectedType, GetStateAllocator()).Move(), GetStateAllocator());
    }
    
    // 结束不允许的类型验证，创建错误对象，添加期望的类型和实际类型到错误对象中，将当前错误设置为错误对象，并添加当前错误类型
    void EndDisallowedType(const typename SchemaType::ValueType& actualType) {
        ValueType error(kObjectType);
        error.AddMember(GetExpectedString(), currentError_, GetStateAllocator());
        error.AddMember(GetActualString(), ValueType(actualType, GetStateAllocator()).Move(), GetStateAllocator());
        currentError_ = error;
        AddCurrentError(kValidateErrorType);
    }
    
    // 不是所有的验证都通过，将错误类型设置为 kValidateErrorAllOf，并添加错误数组
    void NotAllOf(ISchemaValidator** subvalidators, SizeType count) {
        // 将 allOf 视为 oneOf 和 anyOf 来匹配 https://rapidjson.org/md_doc_schema.html#allOf-anyOf-oneOf
        AddErrorArray(kValidateErrorAllOf, subvalidators, count);
        //for (SizeType i = 0; i < count; ++i) {
        //    MergeError(static_cast<GenericSchemaValidator*>(subvalidators[i])->GetError());
        //}
    }
    
    // 没有一个验证通过，将错误类型设置为 kValidateErrorAnyOf，并添加错误数组
    void NoneOf(ISchemaValidator** subvalidators, SizeType count) {
        AddErrorArray(kValidateErrorAnyOf, subvalidators, count);
    }
    
    // 不是一个验证通过，根据 matched 参数设置错误类型为 kValidateErrorOneOfMatch 或 kValidateErrorOneOf，并添加错误数组
    void NotOneOf(ISchemaValidator** subvalidators, SizeType count, bool matched = false) {
        AddErrorArray(matched ? kValidateErrorOneOfMatch : kValidateErrorOneOf, subvalidators, count);
    }
    
    // 不允许的类型，将当前错误设置为对象，并添加当前错误类型
    void Disallowed() {
        currentError_.SetObject();
        AddCurrentError(kValidateErrorNot);
    }
# 定义一个宏，用于生成静态字符串常量和对应的获取函数
#define RAPIDJSON_STRING_(name, ...) \
    # 生成静态字符串常量的获取函数
    static const StringRefType& Get##name##String() {\
        # 生成静态字符数组
        static const Ch s[] = { __VA_ARGS__, '\0' };\
        # 生成字符串引用对象
        static const StringRefType v(s, static_cast<SizeType>(sizeof(s) / sizeof(Ch) - 1)); \
        return v;\
    }

# 使用宏生成具体的字符串常量和对应的获取函数
RAPIDJSON_STRING_(InstanceRef, 'i', 'n', 's', 't', 'a', 'n', 'c', 'e', 'R', 'e', 'f')
RAPIDJSON_STRING_(SchemaRef, 's', 'c', 'h', 'e', 'm', 'a', 'R', 'e', 'f')
RAPIDJSON_STRING_(Expected, 'e', 'x', 'p', 'e', 'c', 't', 'e', 'd')
RAPIDJSON_STRING_(Actual, 'a', 'c', 't', 'u', 'a', 'l')
RAPIDJSON_STRING_(Disallowed, 'd', 'i', 's', 'a', 'l', 'l', 'o', 'w', 'e', 'd')
RAPIDJSON_STRING_(Missing, 'm', 'i', 's', 's', 'i', 'n', 'g')
RAPIDJSON_STRING_(Errors, 'e', 'r', 'r', 'o', 'r', 's')
RAPIDJSON_STRING_(ErrorCode, 'e', 'r', 'r', 'o', 'r', 'C', 'o', 'd', 'e')
RAPIDJSON_STRING_(ErrorMessage, 'e', 'r', 'r', 'o', 'r', 'M', 'e', 's', 's', 'a', 'g', 'e')
RAPIDJSON_STRING_(Duplicates, 'd', 'u', 'p', 'l', 'i', 'c', 'a', 't', 'e', 's')

# 取消宏定义
#undef RAPIDJSON_STRING_

# 如果启用了详细模式，则定义处理开始的宏
#if RAPIDJSON_SCHEMA_VERBOSE
# 定义处理开始的宏
#define RAPIDJSON_SCHEMA_HANDLE_BEGIN_VERBOSE_() \
RAPIDJSON_MULTILINEMACRO_BEGIN\
    # 将空字符压入文档栈
    *documentStack_.template Push<Ch>() = '\0';\
    # 从文档栈中弹出一个字符
    documentStack_.template Pop<Ch>(1);\
    # 打印无效文档的错误信息
    internal::PrintInvalidDocument(documentStack_.template Bottom<Ch>());\
RAPIDJSON_MULTILINEMACRO_END
#else
# 如果未启用详细模式，则定义为空的处理开始的宏
#define RAPIDJSON_SCHEMA_HANDLE_BEGIN_VERBOSE_()
#endif

# 定义处理开始的宏，包括方法和参数
#define RAPIDJSON_SCHEMA_HANDLE_BEGIN_(method, arg1)\
    # 如果验证无效，则返回false
    if (!valid_) return false; \
    # 如果不是值的开始，并且不继续在错误时继续，则返回无效
    if ((!BeginValue() && !GetContinueOnErrors()) || (!CurrentSchema().method arg1 && !GetContinueOnErrors())) {\
        # 调用处理开始详细的宏
        RAPIDJSON_SCHEMA_HANDLE_BEGIN_VERBOSE_();\
        # 返回验证结果为false
        return valid_ = false;\
    }

# 定义并行处理的宏，包括方法和参数
#define RAPIDJSON_SCHEMA_HANDLE_PARALLEL_(method, arg2)\
    # 遍历 schemaStack_ 中的 Context 对象
    for (Context* context = schemaStack_.template Bottom<Context>(); context != schemaStack_.template End<Context>(); context++) {\
        # 如果 Context 对象中存在哈希器，则调用哈希器的指定方法
        if (context->hasher)\
            static_cast<HasherType*>(context->hasher)->method arg2;\
        # 如果 Context 对象中存在验证器，则遍历并调用每个验证器的指定方法
        if (context->validators)\
            for (SizeType i_ = 0; i_ < context->validatorCount; i_++)\
                static_cast<GenericSchemaValidator*>(context->validators[i_])->method arg2;\
        # 如果 Context 对象中存在模式属性验证器，则遍历并调用每个模式属性验证器的指定方法
        if (context->patternPropertiesValidators)\
            for (SizeType i_ = 0; i_ < context->patternPropertiesValidatorCount; i_++)\
                static_cast<GenericSchemaValidator*>(context->patternPropertiesValidators[i_])->method arg2;\
    }
# 定义一个宏，用于处理结束值的情况
#define RAPIDJSON_SCHEMA_HANDLE_END_(method, arg2)\
    valid_ = (EndValue() || GetContinueOnErrors()) && (!outputHandler_ || outputHandler_->method arg2);\
    return valid_;

# 定义一个宏，用于处理值的情况
#define RAPIDJSON_SCHEMA_HANDLE_VALUE_(method, arg1, arg2) \
    RAPIDJSON_SCHEMA_HANDLE_BEGIN_   (method, arg1);\
    RAPIDJSON_SCHEMA_HANDLE_PARALLEL_(method, arg2);\
    RAPIDJSON_SCHEMA_HANDLE_END_     (method, arg2)

# 处理空值的情况
bool Null()             { RAPIDJSON_SCHEMA_HANDLE_VALUE_(Null,   (CurrentContext()), ( )); }
# 处理布尔值的情况
bool Bool(bool b)       { RAPIDJSON_SCHEMA_HANDLE_VALUE_(Bool,   (CurrentContext(), b), (b)); }
# 处理整数值的情况
bool Int(int i)         { RAPIDJSON_SCHEMA_HANDLE_VALUE_(Int,    (CurrentContext(), i), (i)); }
# 处理无符号整数值的情况
bool Uint(unsigned u)   { RAPIDJSON_SCHEMA_HANDLE_VALUE_(Uint,   (CurrentContext(), u), (u)); }
# 处理64位整数值的情况
bool Int64(int64_t i)   { RAPIDJSON_SCHEMA_HANDLE_VALUE_(Int64,  (CurrentContext(), i), (i)); }
# 处理64位无符号整数值的情况
bool Uint64(uint64_t u) { RAPIDJSON_SCHEMA_HANDLE_VALUE_(Uint64, (CurrentContext(), u), (u)); }
# 处理双精度浮点数值的情况
bool Double(double d)   { RAPIDJSON_SCHEMA_HANDLE_VALUE_(Double, (CurrentContext(), d), (d)); }
# 处理原始数字字符串的情况
bool RawNumber(const Ch* str, SizeType length, bool copy)
                                    { RAPIDJSON_SCHEMA_HANDLE_VALUE_(String, (CurrentContext(), str, length, copy), (str, length, copy)); }
# 处理字符串的情况
bool String(const Ch* str, SizeType length, bool copy)
                                    { RAPIDJSON_SCHEMA_HANDLE_VALUE_(String, (CurrentContext(), str, length, copy), (str, length, copy)); }

# 处理开始对象的情况
bool StartObject() {
    RAPIDJSON_SCHEMA_HANDLE_BEGIN_(StartObject, (CurrentContext()));
    RAPIDJSON_SCHEMA_HANDLE_PARALLEL_(StartObject, ());
    return valid_ = !outputHandler_ || outputHandler_->StartObject();
}
    # 检查键是否有效，如果无效则返回 false
    bool Key(const Ch* str, SizeType len, bool copy) {
        # 如果键无效，则返回 false
        if (!valid_) return false;
        # 将键添加到令牌中
        AppendToken(str, len);
        # 如果当前模式中没有该键，并且不允许继续出现错误，则将 valid_ 设置为 false
        if (!CurrentSchema().Key(CurrentContext(), str, len, copy) && !GetContinueOnErrors()) return valid_ = false;
        # 并行处理键
        RAPIDJSON_SCHEMA_HANDLE_PARALLEL_(Key, (str, len, copy));
        # 如果没有输出处理程序，或者输出处理程序的键方法返回 true，则将 valid_ 设置为 true
        return valid_ = !outputHandler_ || outputHandler_->Key(str, len, copy);
    }
    
    # 结束对象的处理
    bool EndObject(SizeType memberCount) {
        # 如果无效，则返回 false
        if (!valid_) return false;
        # 并行处理结束对象
        RAPIDJSON_SCHEMA_HANDLE_PARALLEL_(EndObject, (memberCount));
        # 如果当前模式中结束对象失败，并且不允许继续出现错误，则将 valid_ 设置为 false
        if (!CurrentSchema().EndObject(CurrentContext(), memberCount) && !GetContinueOnErrors()) return valid_ = false;
        # 结束对象处理
        RAPIDJSON_SCHEMA_HANDLE_END_(EndObject, (memberCount));
    }
    
    # 开始数组处理
    bool StartArray() {
        # 开始数组处理的开始
        RAPIDJSON_SCHEMA_HANDLE_BEGIN_(StartArray, (CurrentContext()));
        # 并行处理开始数组
        RAPIDJSON_SCHEMA_HANDLE_PARALLEL_(StartArray, ());
        # 如果没有输出处理程序，或者输出处理程序的开始数组方法返回 true，则将 valid_ 设置为 true
        return valid_ = !outputHandler_ || outputHandler_->StartArray();
    }
    
    # 结束数组处理
    bool EndArray(SizeType elementCount) {
        # 如果无效，则返回 false
        if (!valid_) return false;
        # 并行处理结束数组
        RAPIDJSON_SCHEMA_HANDLE_PARALLEL_(EndArray, (elementCount));
        # 如果当前模式中结束数组失败，并且不允许继续出现错误，则将 valid_ 设置为 false
        if (!CurrentSchema().EndArray(CurrentContext(), elementCount) && !GetContinueOnErrors()) return valid_ = false;
        # 结束数组处理
        RAPIDJSON_SCHEMA_HANDLE_END_(EndArray, (elementCount));
    }
// 取消定义 RAPIDJSON_SCHEMA_HANDLE_BEGIN_VERBOSE_
#undef RAPIDJSON_SCHEMA_HANDLE_BEGIN_VERBOSE_
// 取消定义 RAPIDJSON_SCHEMA_HANDLE_BEGIN_
#undef RAPIDJSON_SCHEMA_HANDLE_BEGIN_
// 取消定义 RAPIDJSON_SCHEMA_HANDLE_PARALLEL_
#undef RAPIDJSON_SCHEMA_HANDLE_PARALLEL_
// 取消定义 RAPIDJSON_SCHEMA_HANDLE_VALUE_
#undef RAPIDJSON_SCHEMA_HANDLE_VALUE_

// 实现 ISchemaStateFactory<SchemaType> 接口
virtual ISchemaValidator* CreateSchemaValidator(const SchemaType& root, const bool inheritContinueOnErrors) {
    // 使用状态分配器分配内存并创建 GenericSchemaValidator 对象
    ISchemaValidator* sv = new (GetStateAllocator().Malloc(sizeof(GenericSchemaValidator))) GenericSchemaValidator(*schemaDocument_, root, documentStack_.template Bottom<char>(), documentStack_.GetSize(),
#if RAPIDJSON_SCHEMA_VERBOSE
    depth_ + 1,
#endif
    &GetStateAllocator());
    // 设置验证标志
    sv->SetValidateFlags(inheritContinueOnErrors ? GetValidateFlags() : GetValidateFlags() & ~(unsigned)kValidateContinueOnErrorFlag);
    return sv;
}

// 销毁 ISchemaValidator 对象
virtual void DestroySchemaValidator(ISchemaValidator* validator) {
    GenericSchemaValidator* v = static_cast<GenericSchemaValidator*>(validator);
    v->~GenericSchemaValidator();
    StateAllocator::Free(v);
}

// 创建哈希器对象
virtual void* CreateHasher() {
    return new (GetStateAllocator().Malloc(sizeof(HasherType))) HasherType(&GetStateAllocator());
}

// 获取哈希值
virtual uint64_t GetHashCode(void* hasher) {
    return static_cast<HasherType*>(hasher)->GetHashCode();
}

// 销毁哈希器对象
virtual void DestroryHasher(void* hasher) {
    HasherType* h = static_cast<HasherType*>(hasher);
    h->~HasherType();
    StateAllocator::Free(h);
}

// 分配状态内存
virtual void* MallocState(size_t size) {
    return GetStateAllocator().Malloc(size);
}

// 释放状态内存
virtual void FreeState(void* p) {
    StateAllocator::Free(p);
}

private:
    // 定义上下文和哈希码数组类型
    typedef typename SchemaType::Context Context;
    typedef GenericValue<UTF8<>, StateAllocator> HashCodeArray;
    // 定义哈希器类型
    typedef internal::Hasher<EncodingType, StateAllocator> HasherType;
    # 创建一个通用的模式验证器对象
    GenericSchemaValidator(
        # 传入模式文档类型的引用
        const SchemaDocumentType& schemaDocument,
        # 传入根模式类型的引用
        const SchemaType& root,
        # 传入基本路径的指针和大小
        const char* basePath, size_t basePathSize,
#if RAPIDJSON_SCHEMA_VERBOSE
        unsigned depth,  // 如果定义了 RAPIDJSON_SCHEMA_VERBOSE，则声明一个无符号整数变量 depth
#endif
        StateAllocator* allocator = 0,  // 初始化一个 StateAllocator 指针 allocator，赋值为 0
        size_t schemaStackCapacity = kDefaultSchemaStackCapacity,  // 初始化一个 size_t 类型的变量 schemaStackCapacity，赋值为 kDefaultSchemaStackCapacity
        size_t documentStackCapacity = kDefaultDocumentStackCapacity)  // 初始化一个 size_t 类型的变量 documentStackCapacity，赋值为 kDefaultDocumentStackCapacity
        :
        schemaDocument_(&schemaDocument),  // 初始化 schemaDocument_ 指针，指向 schemaDocument
        root_(root),  // 初始化 root_ 指针，指向 root
        stateAllocator_(allocator),  // 初始化 stateAllocator_ 指针，指向 allocator
        ownStateAllocator_(0),  // 初始化 ownStateAllocator_，赋值为 0
        schemaStack_(allocator, schemaStackCapacity),  // 使用 allocator 和 schemaStackCapacity 初始化 schemaStack_
        documentStack_(allocator, documentStackCapacity),  // 使用 allocator 和 documentStackCapacity 初始化 documentStack_
        outputHandler_(0),  // 初始化 outputHandler_，赋值为 0
        error_(kObjectType),  // 初始化 error_，类型为 kObjectType
        currentError_(),  // 初始化 currentError_
        missingDependents_(),  // 初始化 missingDependents_
        valid_(true),  // 初始化 valid_，赋值为 true
        flags_(kValidateDefaultFlags)  // 初始化 flags_，赋值为 kValidateDefaultFlags
#if RAPIDJSON_SCHEMA_VERBOSE
        , depth_(depth)  // 如果定义了 RAPIDJSON_SCHEMA_VERBOSE，则初始化 depth_
#endif
    {
        if (basePath && basePathSize)  // 如果 basePath 和 basePathSize 都存在
            memcpy(documentStack_.template Push<char>(basePathSize), basePath, basePathSize);  // 使用 memcpy 将 basePath 的内容复制到 documentStack_
    }

    StateAllocator& GetStateAllocator() {  // 获取 StateAllocator
        if (!stateAllocator_)  // 如果 stateAllocator_ 不存在
            stateAllocator_ = ownStateAllocator_ = RAPIDJSON_NEW(StateAllocator)();  // 创建一个新的 StateAllocator，并赋值给 stateAllocator_ 和 ownStateAllocator_
        return *stateAllocator_;  // 返回 stateAllocator_ 的引用
    }

    bool GetContinueOnErrors() const {  // 获取是否继续在错误发生时继续
        return flags_ & kValidateContinueOnErrorFlag;  // 返回 flags_ 是否包含 kValidateContinueOnErrorFlag 标志
    }
    // 开始处理值的解析
    bool BeginValue() {
        // 如果 schemaStack_ 为空，则将 root_ 压入栈中
        if (schemaStack_.Empty())
            PushSchema(root_);
        else {
            // 如果当前上下文在数组中，则将当前数组元素索引添加到 documentStack_ 中
            if (CurrentContext().inArray)
                internal::TokenHelper<internal::Stack<StateAllocator>, Ch>::AppendIndexToken(documentStack_, CurrentContext().arrayElementIndex);

            // 如果当前 schema 的 BeginValue 返回 false，并且不允许继续错误，则返回 false
            if (!CurrentSchema().BeginValue(CurrentContext()) && !GetContinueOnErrors())
                return false;

            // 获取当前上下文中的 patternPropertiesSchemaCount、patternPropertiesSchemas、valuePatternValidatorType、valueUniqueness
            SizeType count = CurrentContext().patternPropertiesSchemaCount;
            const SchemaType** sa = CurrentContext().patternPropertiesSchemas;
            typename Context::PatternValidatorType patternValidatorType = CurrentContext().valuePatternValidatorType;
            bool valueUniqueness = CurrentContext().valueUniqueness;
            RAPIDJSON_ASSERT(CurrentContext().valueSchema);
            // 将当前 valueSchema 压入栈中
            PushSchema(*CurrentContext().valueSchema);

            // 如果 count 大于 0，则处理 patternPropertiesValidators
            if (count > 0) {
                CurrentContext().objectPatternValidatorType = patternValidatorType;
                ISchemaValidator**& va = CurrentContext().patternPropertiesValidators;
                SizeType& validatorCount = CurrentContext().patternPropertiesValidatorCount;
                va = static_cast<ISchemaValidator**>(MallocState(sizeof(ISchemaValidator*) * count));
                for (SizeType i = 0; i < count; i++)
                    va[validatorCount++] = CreateSchemaValidator(*sa[i], true);  // 继承 continueOnError
            }

            // 设置当前上下文中的 arrayUniqueness 为 valueUniqueness
            CurrentContext().arrayUniqueness = valueUniqueness;
        }
        return true;
    }

    // 结束值的解析
    bool EndValue() {
        // 如果当前 schema 的 EndValue 返回 false，并且不允许继续错误，则返回 false
        if (!CurrentSchema().EndValue(CurrentContext()) && !GetContinueOnErrors())
            return false;
#if RAPIDJSON_SCHEMA_VERBOSE
        // 如果定义了 RAPIDJSON_SCHEMA_VERBOSE 宏，则执行以下代码块
        GenericStringBuffer<EncodingType> sb;
        // 从 schemaDocument_ 获取当前模式的指针，并将其转换为字符串形式存储在 sb 中
        schemaDocument_->GetPointer(&CurrentSchema()).Stringify(sb);

        // 将一个空字符压入 documentStack_ 中
        *documentStack_.template Push<Ch>() = '\0';
        // 从 documentStack_ 中弹出一个字符
        documentStack_.template Pop<Ch>(1);
        // 打印验证器指针的信息
        internal::PrintValidatorPointers(depth_, sb.GetString(), documentStack_.template Bottom<Ch>());
#endif
        // 获取当前上下文的哈希器
        void* hasher = CurrentContext().hasher;
        // 如果哈希器存在并且当前上下文的数组唯一性为真，则计算哈希值
        uint64_t h = hasher && CurrentContext().arrayUniqueness ? static_cast<HasherType*>(hasher)->GetHashCode() : 0;

        // 弹出当前模式
        PopSchema();

        // 如果模式栈不为空，则执行以下代码块
        if (!schemaStack_.Empty()) {
            Context& context = CurrentContext();
            // 只有在存在哈希器的情况下才检查唯一性
            if (hasher && context.valueUniqueness) {
                HashCodeArray* a = static_cast<HashCodeArray*>(context.arrayElementHashCodes);
                // 如果哈希码数组不存在，则创建一个新的哈希码数组
                if (!a)
                    CurrentContext().arrayElementHashCodes = a = new (GetStateAllocator().Malloc(sizeof(HashCodeArray))) HashCodeArray(kArrayType);
                // 遍历哈希码数组，检查是否存在重复的哈希值
                for (typename HashCodeArray::ConstValueIterator itr = a->Begin(); itr != a->End(); ++itr)
                    if (itr->GetUint64() == h) {
                        // 如果存在重复的哈希值，则执行以下代码块
                        DuplicateItems(static_cast<SizeType>(itr - a->Begin()), a->Size());
                        // 如果设置了继续错误标志，则在返回之前进行清理
                        if (GetContinueOnErrors()) {
                            a->PushBack(h, GetStateAllocator());
                            while (!documentStack_.Empty() && *documentStack_.template Pop<Ch>(1) != '/');
                        }
                        // 返回验证错误唯一项
                        RAPIDJSON_INVALID_KEYWORD_RETURN(kValidateErrorUniqueItems);
                    }
                // 将当前哈希值压入哈希码数组
                a->PushBack(h, GetStateAllocator());
            }
        }

        // 移除文档指针的最后一个标记
        while (!documentStack_.Empty() && *documentStack_.template Pop<Ch>(1) != '/')
            ;

        // 返回 true
        return true;
    }
    // 向文档栈中添加一个令牌
    void AppendToken(const Ch* str, SizeType len) {
        // 预留足够的空间，最坏情况下所有字符都被转义为两个字符
        documentStack_.template Reserve<Ch>(1 + len * 2);
        // 在文档栈中添加 '/'
        *documentStack_.template PushUnsafe<Ch>() = '/';
        for (SizeType i = 0; i < len; i++) {
            // 如果字符为 '~'，则在文档栈中添加 '~' 和 '0'
            if (str[i] == '~') {
                *documentStack_.template PushUnsafe<Ch>() = '~';
                *documentStack_.template PushUnsafe<Ch>() = '0';
            }
            // 如果字符为 '/'，则在文档栈中添加 '~' 和 '1'
            else if (str[i] == '/') {
                *documentStack_.template PushUnsafe<Ch>() = '~';
                *documentStack_.template PushUnsafe<Ch>() = '1';
            }
            // 否则直接在文档栈中添加字符
            else
                *documentStack_.template PushUnsafe<Ch>() = str[i];
        }
    }

    // 将模式推入模式栈
    RAPIDJSON_FORCEINLINE void PushSchema(const SchemaType& schema) { new (schemaStack_.template Push<Context>()) Context(*this, *this, &schema); }

    // 从模式栈中弹出模式
    RAPIDJSON_FORCEINLINE void PopSchema() {
        Context* c = schemaStack_.template Pop<Context>(1);
        if (HashCodeArray* a = static_cast<HashCodeArray*>(c->arrayElementHashCodes)) {
            a->~HashCodeArray();
            StateAllocator::Free(a);
        }
        c->~Context();
    }

    // 添加错误实例的位置
    void AddErrorInstanceLocation(ValueType& result, bool parent) {
        // 创建字符串缓冲区
        GenericStringBuffer<EncodingType> sb;
        // 获取无效文档指针
        PointerType instancePointer = GetInvalidDocumentPointer();
        // 将实例指针转换为 URI 片段并添加到结果中
        ((parent && instancePointer.GetTokenCount() > 0)
         ? PointerType(instancePointer.GetTokens(), instancePointer.GetTokenCount() - 1)
         : instancePointer).StringifyUriFragment(sb);
        ValueType instanceRef(sb.GetString(), static_cast<SizeType>(sb.GetSize() / sizeof(Ch)),
                              GetStateAllocator());
        result.AddMember(GetInstanceRefString(), instanceRef, GetStateAllocator());
    }
    // 向结果对象中添加错误模式的位置信息
    void AddErrorSchemaLocation(ValueType& result, PointerType schema = PointerType()) {
        // 创建字符串缓冲区
        GenericStringBuffer<EncodingType> sb;
        // 获取当前模式的URI长度
        SizeType len = CurrentSchema().GetURI().GetStringLength();
        // 如果长度大于0，则将URI字符串复制到缓冲区中
        if (len) memcpy(sb.Push(len), CurrentSchema().GetURI().GetString(), len * sizeof(Ch));
        // 如果模式有标记计数，则将其字符串化为URI片段
        if (schema.GetTokenCount()) schema.StringifyUriFragment(sb);
        // 否则将无效的模式指针字符串化为URI片段
        else GetInvalidSchemaPointer().StringifyUriFragment(sb);
        // 创建包含模式引用的值对象
        ValueType schemaRef(sb.GetString(), static_cast<SizeType>(sb.GetSize() / sizeof(Ch)),
            GetStateAllocator());
        // 向结果对象中添加模式引用
        result.AddMember(GetSchemaRefString(), schemaRef, GetStateAllocator());
    }

    // 向结果对象中添加错误代码
    void AddErrorCode(ValueType& result, const ValidateErrorCode code) {
        // 向结果对象中添加错误代码
        result.AddMember(GetErrorCodeString(), code, GetStateAllocator());
    }

    // 向关键字对象中添加错误
    void AddError(ValueType& keyword, ValueType& error) {
        // 查找关键字对应的错误成员
        typename ValueType::MemberIterator member = error_.FindMember(keyword);
        // 如果未找到，则添加新的错误成员
        if (member == error_.MemberEnd())
            error_.AddMember(keyword, error, GetStateAllocator());
        // 如果找到了，则根据情况进行处理
        else {
            // 如果值是对象，则创建新的错误数组，将原值放入数组中
            if (member->value.IsObject()) {
                ValueType errors(kArrayType);
                errors.PushBack(member->value, GetStateAllocator());
                member->value = errors;
            }
            // 将新的错误添加到原值中
            member->value.PushBack(error, GetStateAllocator());
        }
    }

    // 向当前错误对象中添加错误代码和位置信息
    void AddCurrentError(const ValidateErrorCode code, bool parent = false) {
        // 向当前错误对象中添加错误代码
        AddErrorCode(currentError_, code);
        // 向当前错误对象中添加错误实例位置信息
        AddErrorInstanceLocation(currentError_, parent);
        // 向当前错误对象中添加错误模式位置信息
        AddErrorSchemaLocation(currentError_);
        // 向错误对象中添加当前错误
        AddError(ValueType(SchemaType::GetValidateErrorKeyword(code), GetStateAllocator(), false).Move(), currentError_);
    }

    // 合并错误对象
    void MergeError(ValueType& other) {
        // 遍历其他错误对象的成员，并添加到当前错误对象中
        for (typename ValueType::MemberIterator it = other.MemberBegin(), end = other.MemberEnd(); it != end; ++it) {
            AddError(it->name, it->value);
        }
    }
    // 向错误对象中添加错误码、实际值、期望值和可选的exclusive值
    void AddNumberError(const ValidateErrorCode code, ValueType& actual, const SValue& expected,
        const typename SchemaType::ValueType& (*exclusive)() = 0) {
        currentError_.SetObject();
        currentError_.AddMember(GetActualString(), actual, GetStateAllocator());
        currentError_.AddMember(GetExpectedString(), ValueType(expected, GetStateAllocator()).Move(), GetStateAllocator());
        if (exclusive)
            currentError_.AddMember(ValueType(exclusive(), GetStateAllocator()).Move(), true, GetStateAllocator());
        AddCurrentError(code);
    }

    // 向错误对象中添加错误码和子验证器的错误
    void AddErrorArray(const ValidateErrorCode code,
        ISchemaValidator** subvalidators, SizeType count) {
        ValueType errors(kArrayType);
        for (SizeType i = 0; i < count; ++i)
            errors.PushBack(static_cast<GenericSchemaValidator*>(subvalidators[i])->GetError(), GetStateAllocator());
        currentError_.SetObject();
        currentError_.AddMember(GetErrorsString(), errors, GetStateAllocator());
        AddCurrentError(code);
    }

    // 返回当前模式
    const SchemaType& CurrentSchema() const { return *schemaStack_.template Top<Context>()->schema; }
    // 返回当前上下文
    Context& CurrentContext() { return *schemaStack_.template Top<Context>(); }
    // 返回当前上下文（const版本）
    const Context& CurrentContext() const { return *schemaStack_.template Top<Context>(); }

    // 默认模式堆栈容量
    static const size_t kDefaultSchemaStackCapacity = 1024;
    // 默认文档堆栈容量
    static const size_t kDefaultDocumentStackCapacity = 256;
    const SchemaDocumentType* schemaDocument_;
    const SchemaType& root_;
    StateAllocator* stateAllocator_;
    StateAllocator* ownStateAllocator_;
    internal::Stack<StateAllocator> schemaStack_;    //!< 用于存储当前模式路径的堆栈（BaseSchemaType *）
    internal::Stack<StateAllocator> documentStack_;  //!< 用于存储当前验证文档路径的堆栈（Ch）
    OutputHandler* outputHandler_;
    ValueType error_;
    ValueType currentError_;
    ValueType missingDependents_;
    bool valid_;
    unsigned flags_;
#if RAPIDJSON_SCHEMA_VERBOSE
    // 如果定义了 RAPIDJSON_SCHEMA_VERBOSE，则定义一个无符号整数变量 depth_
    unsigned depth_;
#endif
};

// 定义一个模板类 SchemaValidator，继承自 GenericSchemaValidator，模板参数为 SchemaDocument
typedef GenericSchemaValidator<SchemaDocument> SchemaValidator;

///////////////////////////////////////////////////////////////////////////////
// SchemaValidatingReader

//! 用于带有验证功能的解析的辅助类。
/*!
    这个辅助类是一个函数对象，设计为 \ref GenericDocument::Populate() 的参数。

    \tparam parseFlags 组合了 \ref ParseFlag。
    \tparam InputStream 输入流的类型，实现了 Stream 概念。
    \tparam SourceEncoding 输入流的编码。
    \tparam SchemaDocumentType 模式文档的类型。
    \tparam StackAllocator 用于堆栈的分配器类型。
*/
template <
    unsigned parseFlags,
    typename InputStream,
    typename SourceEncoding,
    typename SchemaDocumentType = SchemaDocument,
    typename StackAllocator = CrtAllocator>
class SchemaValidatingReader {
public:
    // 定义指针类型为 SchemaDocumentType 的 PointerType
    typedef typename SchemaDocumentType::PointerType PointerType;
    // 定义输入流的字符类型为 Ch
    typedef typename InputStream::Ch Ch;
    // 定义值类型为 GenericValue，模板参数为 SourceEncoding 和 StackAllocator
    typedef GenericValue<SourceEncoding, StackAllocator> ValueType;

    //! 构造函数
    /*!
        \param is 输入流。
        \param sd 模式文档。
    */
    SchemaValidatingReader(InputStream& is, const SchemaDocumentType& sd) : is_(is), sd_(sd), invalidSchemaKeyword_(), invalidSchemaCode_(kValidateErrorNone), error_(kObjectType), isValid_(true) {}

    template <typename Handler>
    # 重载函数调用操作符，接受一个处理器对象作为参数
    bool operator()(Handler& handler) {
        # 创建一个通用的读取器对象，使用给定的编码和分配器
        GenericReader<SourceEncoding, typename SchemaDocumentType::EncodingType, StackAllocator> reader;
        # 创建一个通用的模式验证器对象，使用给定的模式文档和处理器
        GenericSchemaValidator<SchemaDocumentType, Handler> validator(sd_, handler);
        # 使用读取器解析输入流，并使用验证器进行验证
        parseResult_ = reader.template Parse<parseFlags>(is_, validator);

        # 检查验证器是否有效
        isValid_ = validator.IsValid();
        # 如果验证通过，重置错误信息
        if (isValid_) {
            invalidSchemaPointer_ = PointerType();
            invalidSchemaKeyword_ = 0;
            invalidDocumentPointer_ = PointerType();
            error_.SetObject();
        }
        # 如果验证失败，获取错误信息
        else {
            invalidSchemaPointer_ = validator.GetInvalidSchemaPointer();
            invalidSchemaKeyword_ = validator.GetInvalidSchemaKeyword();
            invalidSchemaCode_ = validator.GetInvalidSchemaCode();
            invalidDocumentPointer_ = validator.GetInvalidDocumentPointer();
            error_.CopyFrom(validator.GetError(), allocator_);
        }

        # 返回解析结果
        return parseResult_;
    }

    # 获取解析结果
    const ParseResult& GetParseResult() const { return parseResult_; }
    # 检查验证是否通过
    bool IsValid() const { return isValid_; }
    # 获取无效模式指针
    const PointerType& GetInvalidSchemaPointer() const { return invalidSchemaPointer_; }
    # 获取无效模式关键字
    const Ch* GetInvalidSchemaKeyword() const { return invalidSchemaKeyword_; }
    # 获取无效文档指针
    const PointerType& GetInvalidDocumentPointer() const { return invalidDocumentPointer_; }
    # 获取错误信息
    const ValueType& GetError() const { return error_; }
    # 获取无效模式代码
    ValidateErrorCode GetInvalidSchemaCode() const { return invalidSchemaCode_; }
// 声明私有成员变量，输入流和模式文档类型的引用
private:
    InputStream& is_;
    const SchemaDocumentType& sd_;

    // 解析结果
    ParseResult parseResult_;
    // 无效模式指针
    PointerType invalidSchemaPointer_;
    // 无效模式关键字
    const Ch* invalidSchemaKeyword_;
    // 无效文档指针
    PointerType invalidDocumentPointer_;
    // 无效模式错误码
    ValidateErrorCode invalidSchemaCode_;
    // 分配器
    StackAllocator allocator_;
    // 错误值类型
    ValueType error_;
    // 是否有效
    bool isValid_;
};

// 结束 RapidJSON 命名空间
RAPIDJSON_NAMESPACE_END
// 弹出 RapidJSON 的诊断状态
RAPIDJSON_DIAG_POP

// 结束 RapidJSON_SCHEMA_H_ 的条件编译
#endif // RAPIDJSON_SCHEMA_H_
```