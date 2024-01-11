# `xmrig\src\3rdparty\rapidjson\internal\pow10.h`

```
#ifndef RAPIDJSON_POW10_
#define RAPIDJSON_POW10_

#include "../rapidjson.h"

RAPIDJSON_NAMESPACE_BEGIN
namespace internal {

//! 计算 double 类型的 10 的整数次幂 (10.0^n)。
/*! 该函数使用查找表以获得快速和准确的结果。
    \param n 非负指数。必须 <= 308。
    \return 10.0^n
*/
inline double Pow10(int n) {
    // 断言指数 n 大于等于 0 并且小于等于 308
    RAPIDJSON_ASSERT(n >= 0 && n <= 308);
    // 返回查找表中指数对应的值
    return e[n];
}

} // namespace internal
RAPIDJSON_NAMESPACE_END

#endif // RAPIDJSON_POW10_
```