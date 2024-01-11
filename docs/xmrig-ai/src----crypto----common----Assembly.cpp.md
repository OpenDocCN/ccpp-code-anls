# `xmrig\src\crypto\common\Assembly.cpp`

```
/* XMRig
 * 版权所有（c）2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有适销性或特定用途的暗示保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */


#include <cassert>
#include <cstring>


#ifdef _MSC_VER
#   define strcasecmp  _stricmp
#endif


#include "crypto/common/Assembly.h"
#include "3rdparty/rapidjson/document.h"


namespace xmrig {


static const char *asmNames[] = {
    "none",
    "auto",
    "intel",
    "ryzen",
    "bulldozer"
};


} /* namespace xmrig */


// 解析汇编选项的字符串表示形式，返回对应的枚举值
xmrig::Assembly::Id xmrig::Assembly::parse(const char *assembly, Id defaultValue)
{
    // 确定asmNames数组的大小
    constexpr size_t const size = sizeof(asmNames) / sizeof((asmNames)[0]);
    static_assert(size == MAX, "asmNames size mismatch");

    // 如果输入为空，则返回默认值
    if (assembly == nullptr) {
        return defaultValue;
    }

    // 遍历asmNames数组，查找与输入字符串相匹配的枚举值
    for (size_t i = 0; i < size; i++) {
        if (strcasecmp(assembly, asmNames[i]) == 0) {
            return static_cast<Id>(i);
        }
    }

    return defaultValue;
}


// 解析rapidjson::Value类型的汇编选项，返回对应的枚举值
xmrig::Assembly::Id xmrig::Assembly::parse(const rapidjson::Value &value, Id defaultValue)
{
    // 如果值是布尔类型，则返回对应的枚举值
    if (value.IsBool()) {
        return value.GetBool() ? AUTO : NONE;
    }

    // 如果值是字符串类型，则调用上面的parse函数进行解析
    if (value.IsString()) {
        return parse(value.GetString(), defaultValue);
    }

    return defaultValue;
}


// 返回当前枚举值对应的字符串表示
const char *xmrig::Assembly::toString() const
{
    return asmNames[m_id];
}
}
// 将 Assembly 对象转换为 JSON 格式的 rapidjson::Value 对象
rapidjson::Value xmrig::Assembly::toJSON() const
{
    using namespace rapidjson;

    // 如果 m_id 为 NONE，则返回一个布尔值为 false 的 rapidjson::Value 对象
    if (m_id == NONE) {
        return Value(false);
    }

    // 如果 m_id 为 AUTO，则返回一个布尔值为 true 的 rapidjson::Value 对象
    if (m_id == AUTO) {
        return Value(true);
    }

    // 否则，返回一个包含 toString() 返回值的 rapidjson::Value 对象
    return Value(StringRef(toString()));
}
```