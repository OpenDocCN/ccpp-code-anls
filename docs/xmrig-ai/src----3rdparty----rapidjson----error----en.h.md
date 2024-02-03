# `xmrig\src\3rdparty\rapidjson\error\en.h`

```cpp
#ifndef RAPIDJSON_ERROR_EN_H_
#define RAPIDJSON_ERROR_EN_H_

#include "error.h"

#ifdef __clang__
RAPIDJSON_DIAG_PUSH
RAPIDJSON_DIAG_OFF(switch-enum)
RAPIDJSON_DIAG_OFF(covered-switch-default)
#endif

RAPIDJSON_NAMESPACE_BEGIN

//! Maps error code of parsing into error message.
/*!
    \ingroup RAPIDJSON_ERRORS
    \param parseErrorCode Error code obtained in parsing.
    \return the error message.
    \note User can make a copy of this function for localization.
        Using switch-case is safer for future modification of error codes.
*/
inline const RAPIDJSON_ERROR_CHARTYPE* GetParseError_En(ParseErrorCode parseErrorCode) {
    // 将解析错误码映射为错误消息
    // 返回相应的错误消息
    // 用户可以根据需要复制此函数进行本地化
    // 使用 switch-case 更安全，以便将来修改错误码
}

//! Maps error code of validation into error message.
/*!
    \ingroup RAPIDJSON_ERRORS
    \param validateErrorCode Error code obtained from validator.
    \return the error message.
    \note User can make a copy of this function for localization.
        Using switch-case is safer for future modification of error codes.
*/
inline const RAPIDJSON_ERROR_CHARTYPE* GetValidateError_En(ValidateErrorCode validateErrorCode) {
    // 将验证错误码映射为错误消息
    // 返回相应的错误消息
    // 用户可以根据需要复制此函数进行本地化
    // 使用 switch-case 更安全，以便将来修改错误码
}

RAPIDJSON_NAMESPACE_END

#ifdef __clang__
RAPIDJSON_DIAG_POP
#endif

#endif // RAPIDJSON_ERROR_EN_H_
```