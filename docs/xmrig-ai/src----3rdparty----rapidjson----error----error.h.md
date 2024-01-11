# `xmrig\src\3rdparty\rapidjson\error\error.h`

```
// 定义了解析错误码的枚举类型
//! Error code of parsing.
/*! \ingroup RAPIDJSON_ERRORS
    # 参考 GenericReader 类的 Parse 和 GetParseErrorCode 方法
// 定义解析错误代码的枚举类型
enum ParseErrorCode {
    kParseErrorNone = 0,                        //!< No error.

    kParseErrorDocumentEmpty,                   //!< The document is empty.
    kParseErrorDocumentRootNotSingular,         //!< The document root must not follow by other values.

    kParseErrorValueInvalid,                    //!< Invalid value.

    kParseErrorObjectMissName,                  //!< Missing a name for object member.
    kParseErrorObjectMissColon,                 //!< Missing a colon after a name of object member.
    kParseErrorObjectMissCommaOrCurlyBracket,   //!< Missing a comma or '}' after an object member.

    kParseErrorArrayMissCommaOrSquareBracket,   //!< Missing a comma or ']' after an array element.

    kParseErrorStringUnicodeEscapeInvalidHex,   //!< Incorrect hex digit after \\u escape in string.
    kParseErrorStringUnicodeSurrogateInvalid,   //!< The surrogate pair in string is invalid.
    kParseErrorStringEscapeInvalid,             //!< Invalid escape character in string.
    kParseErrorStringMissQuotationMark,         //!< Missing a closing quotation mark in string.
    kParseErrorStringInvalidEncoding,           //!< Invalid encoding in string.

    kParseErrorNumberTooBig,                    //!< Number too big to be stored in double.
    kParseErrorNumberMissFraction,              //!< Miss fraction part in number.
    kParseErrorNumberMissExponent,              //!< Miss exponent in number.

    kParseErrorTermination,                     //!< Parsing was terminated.
    kParseErrorUnspecificSyntaxError            //!< Unspecific syntax error.
};

//! 解析结果（包装 ParseErrorCode）
/*!
    \ingroup RAPIDJSON_ERRORS
    \code
        Document doc;
        ParseResult ok = doc.Parse("[42]");
        if (!ok) {
            fprintf(stderr, "JSON parse error: %s (%u)",
                    GetParseError_En(ok.Code()), ok.Offset());
            exit(EXIT_FAILURE);
        }
    \endcode
    # 参考 GenericReader::Parse 和 GenericDocument::Parse
/*
    结构体 ParseResult，用于表示解析结果
    包含一个未指定的布尔类型
*/
struct ParseResult {
    //!! Unspecified boolean type
    typedef bool (ParseResult::*BooleanType)() const;
public:
    //! 默认构造函数，没有错误
    ParseResult() : code_(kParseErrorNone), offset_(0) {}
    //! 构造函数，设置错误
    ParseResult(ParseErrorCode code, size_t offset) : code_(code), offset_(offset) {}

    //! 获取错误代码
    ParseErrorCode Code() const { return code_; }
    //! 获取错误偏移量，如果是错误则返回偏移量，否则返回0
    size_t Offset() const { return offset_; }

    //! 显式转换为 bool 类型，如果不是错误则返回 true
    operator BooleanType() const { return !IsError() ? &ParseResult::IsError : NULL; }
    //! 判断是否是错误
    bool IsError() const { return code_ != kParseErrorNone; }

    // 重载相等运算符
    bool operator==(const ParseResult& that) const { return code_ == that.code_; }
    bool operator==(ParseErrorCode code) const { return code_ == code; }
    friend bool operator==(ParseErrorCode code, const ParseResult & err) { return code == err.code_; }

    // 重载不等运算符
    bool operator!=(const ParseResult& that) const { return !(*this == that); }
    bool operator!=(ParseErrorCode code) const { return !(*this == code); }
    friend bool operator!=(ParseErrorCode code, const ParseResult & err) { return err != code; }

    //! 重置错误代码
    void Clear() { Set(kParseErrorNone); }
    //! 更新错误代码和偏移量
    void Set(ParseErrorCode code, size_t offset = 0) { code_ = code; offset_ = offset; }

private:
    ParseErrorCode code_; // 错误代码
    size_t offset_; // 偏移量
};

//! 函数指针类型的 GetParseError()
/*! \ingroup RAPIDJSON_ERRORS

    这是 \c GetParseError_X() 的原型，其中 \c X 是一个区域设置。
    用户可以在运行时动态更改区域设置，例如：
\code
    GetParseErrorFunc GetParseError = GetParseError_En; // or whatever
    const RAPIDJSON_ERROR_CHARTYPE* s = GetParseError(document.GetParseErrorCode());
\endcode
*/
// 定义一个函数指针类型，用于获取解析错误信息
typedef const RAPIDJSON_ERROR_CHARTYPE* (*GetParseErrorFunc)(ParseErrorCode);

///////////////////////////////////////////////////////////////////////////////
// ValidateErrorCode

//! 在验证时的错误代码
/*! \ingroup RAPIDJSON_ERRORS
    \see GenericSchemaValidator
*/
enum ValidateErrorCode {
    kValidateErrors    = -1,                   //!< 当设置 kValidateContinueOnErrorsFlag 时的顶层错误代码
    kValidateErrorNone = 0,                    //!< 无错误

    kValidateErrorMultipleOf,                  //!< 数字不是 'multipleOf' 值的倍数
    kValidateErrorMaximum,                     //!< 数字大于 'maximum' 值
    kValidateErrorExclusiveMaximum,            //!< 数字大于或等于 'maximum' 值
    kValidateErrorMinimum,                     //!< 数字小于 'minimum' 值
    kValidateErrorExclusiveMinimum,            //!< 数字小于或等于 'minimum' 值

    kValidateErrorMaxLength,                   //!< 字符串长度大于 'maxLength' 值
    kValidateErrorMinLength,                   //!< 字符串长度小于 'maxLength' 值
    kValidateErrorPattern,                     //!< 字符串不匹配 'pattern' 正则表达式

    kValidateErrorMaxItems,                    //!< 数组长度大于 'maxItems' 值
    kValidateErrorMinItems,                    //!< 数组长度小于 'minItems' 值
    kValidateErrorUniqueItems,                 //!< 数组有重复项，但 'uniqueItems' 为 true
    kValidateErrorAdditionalItems,             //!< 数组有不符合模式的额外项

    kValidateErrorMaxProperties,               //!< 对象成员数大于 'maxProperties' 值
    kValidateErrorMinProperties,               //!< 对象成员数小于 'minProperties' 值
    kValidateErrorRequired,                    //!< 表示对象缺少模式所需的一个或多个成员。
    kValidateErrorAdditionalProperties,        //!< 表示对象具有不被模式允许的额外成员。
    kValidateErrorPatternProperties,           //!< 表示存在其他错误。
    kValidateErrorDependencies,                //!< 表示对象缺少属性或模式依赖项。

    kValidateErrorEnum,                        //!< 表示属性具有不属于其允许枚举值之一的值。
    kValidateErrorType,                        //!< 表示属性具有模式不允许的类型。

    kValidateErrorOneOf,                       //!< 表示属性未匹配由'oneOf'指定的子模式之一。
    kValidateErrorOneOfMatch,                  //!< 表示属性匹配了由'oneOf'指定的多个子模式之一。
    kValidateErrorAllOf,                       //!< 表示属性未匹配由'allOf'指定的所有子模式。
    kValidateErrorAnyOf,                       //!< 表示属性未匹配由'anyOf'指定的任何子模式。
    kValidateErrorNot                          //!< 表示属性匹配了由'not'指定的子模式。
};

//! Function pointer type of GetValidateError().
/*! \ingroup RAPIDJSON_ERRORS

    This is the prototype for \c GetValidateError_X(), where \c X is a locale.
    User can dynamically change locale in runtime, e.g.:
\code
    GetValidateErrorFunc GetValidateError = GetValidateError_En; // or whatever
    const RAPIDJSON_ERROR_CHARTYPE* s = GetValidateError(validator.GetInvalidSchemaCode());
\endcode
*/
typedef const RAPIDJSON_ERROR_CHARTYPE* (*GetValidateErrorFunc)(ValidateErrorCode);

RAPIDJSON_NAMESPACE_END

#ifdef __clang__
RAPIDJSON_DIAG_POP
#endif

#endif // RAPIDJSON_ERROR_ERROR_H_
```