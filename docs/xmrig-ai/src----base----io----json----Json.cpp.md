# `xmrig\src\base\io\json\Json.cpp`

```
/*
 * XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款
 * 它可以是许可证的第3版，也可以是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。更多细节请参见
 * GNU通用公共许可证。
 *
 * 您应该已经收到GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "base/io/json/Json.h"
#include "3rdparty/rapidjson/document.h"


#include <cassert>
#include <cmath>
#include <istream>


namespace xmrig {

static const rapidjson::Value kNullValue;

} // namespace xmrig


// 检查给定的rapidjson::Value对象是否为空
bool xmrig::Json::isEmpty(const rapidjson::Value &obj)
{
    return !obj.IsObject() || obj.ObjectEmpty();
}


// 从rapidjson::Value对象中获取布尔值
bool xmrig::Json::getBool(const rapidjson::Value &obj, const char *key, bool defaultValue)
{
    if (isEmpty(obj)) {
        return defaultValue;
    }

    auto i = obj.FindMember(key);
    if (i != obj.MemberEnd() && i->value.IsBool()) {
        return i->value.GetBool();
    }

    return defaultValue;
}


// 从rapidjson::Value对象中获取字符串
const char *xmrig::Json::getString(const rapidjson::Value &obj, const char *key, const char *defaultValue)
{
    if (isEmpty(obj)) {
        return defaultValue;
    }

    auto i = obj.FindMember(key);
    if (i != obj.MemberEnd() && i->value.IsString()) {
        return i->value.GetString();
    }

    return defaultValue;
}


// 从rapidjson::Value对象中获取数组
const rapidjson::Value &xmrig::Json::getArray(const rapidjson::Value &obj, const char *key)
{
    if (isEmpty(obj)) {
        return kNullValue;
    }

    auto i = obj.FindMember(key);
    # 如果迭代器不指向对象的末尾，并且该对象的值是数组类型
    if (i != obj.MemberEnd() && i->value.IsArray()) {
        # 返回该值
        return i->value;
    }
    # 否则返回空值
    return kNullValue;
// 返回给定 JSON 对象中指定键对应的对象值，如果对象为空则返回空值
const rapidjson::Value &xmrig::Json::getObject(const rapidjson::Value &obj, const char *key)
{
    if (isEmpty(obj)) {
        return kNullValue;
    }

    // 查找指定键对应的成员，并判断其值是否为对象类型，是则返回该值
    auto i = obj.FindMember(key);
    if (i != obj.MemberEnd() && i->value.IsObject()) {
        return i->value;
    }

    // 否则返回空值
    return kNullValue;
}


// 返回给定 JSON 对象中指定键对应的值，如果对象为空则返回空值
const rapidjson::Value &xmrig::Json::getValue(const rapidjson::Value &obj, const char *key)
{
    if (isEmpty(obj)) {
        return kNullValue;
    }

    // 查找指定键对应的成员，并返回其值
    auto i = obj.FindMember(key);
    if (i != obj.MemberEnd()) {
        return i->value;
    }

    // 否则返回空值
    return kNullValue;
}


// 返回给定 JSON 对象中指定键对应的双精度浮点数值，如果对象为空则返回默认值
double xmrig::Json::getDouble(const rapidjson::Value &obj, const char *key, double defaultValue)
{
    if (isEmpty(obj)) {
        return defaultValue;
    }

    // 查找指定键对应的成员，并判断其值是否为双精度浮点数类型，是则返回该值
    auto i = obj.FindMember(key);
    if (i != obj.MemberEnd() && (i->value.IsDouble() || i->value.IsLosslessDouble())) {
        return i->value.GetDouble();
    }

    // 否则返回默认值
    return defaultValue;
}


// 返回给定 JSON 对象中指定键对应的整数值，如果对象为空则返回默认值
int xmrig::Json::getInt(const rapidjson::Value &obj, const char *key, int defaultValue)
{
    if (isEmpty(obj)) {
        return defaultValue;
    }

    // 查找指定键对应的成员，并判断其值是否为整数类型，是则返回该值
    auto i = obj.FindMember(key);
    if (i != obj.MemberEnd() && i->value.IsInt()) {
        return i->value.GetInt();
    }

    // 否则返回默认值
    return defaultValue;
}


// 返回给定 JSON 对象中指定键对应的64位整数值，如果对象为空则返回默认值
int64_t xmrig::Json::getInt64(const rapidjson::Value &obj, const char *key, int64_t defaultValue)
{
    if (isEmpty(obj)) {
        return defaultValue;
    }

    // 查找指定键对应的成员，并判断其值是否为64位整数类型，是则返回该值
    auto i = obj.FindMember(key);
    if (i != obj.MemberEnd() && i->value.IsInt64()) {
        return i->value.GetInt64();
    }

    // 否则返回默认值
    return defaultValue;
}


// 返回给定 JSON 对象中指定键对应的字符串值，如果对象为空则返回空字符串
xmrig::String xmrig::Json::getString(const rapidjson::Value &obj, const char *key, size_t maxSize)
{
    if (isEmpty(obj)) {
        return {};
    }

    // 查找指定键对应的成员，并判断其值是否为字符串类型，是则返回该值
    auto i = obj.FindMember(key);
    if (i == obj.MemberEnd() || !i->value.IsString()) {
        return {};
    }

    // 如果指定了最大长度，并且字符串长度超过最大长度，则返回截断后的字符串
    if (maxSize == 0 || i->value.GetStringLength() <= maxSize) {
        return i->value.GetString();
    }

    // 否则返回截断后的字符串
    return { i->value.GetString(), maxSize };
}
// 从 rapidjson::Value 对象中获取 uint64_t 类型的数值，如果不存在则返回默认值
uint64_t xmrig::Json::getUint64(const rapidjson::Value &obj, const char *key, uint64_t defaultValue)
{
    // 如果对象为空，则返回默认值
    if (isEmpty(obj)) {
        return defaultValue;
    }

    // 查找指定键对应的成员
    auto i = obj.FindMember(key);
    // 如果找到指定键对应的成员，并且其值是 uint64_t 类型，则返回该值
    if (i != obj.MemberEnd() && i->value.IsUint64()) {
        return i->value.GetUint64();
    }

    // 返回默认值
    return defaultValue;
}


// 从 rapidjson::Value 对象中获取 unsigned 类型的数值，如果不存在则返回默认值
unsigned xmrig::Json::getUint(const rapidjson::Value &obj, const char *key, unsigned defaultValue)
{
    // 如果对象为空，则返回默认值
    if (isEmpty(obj)) {
        return defaultValue;
    }

    // 查找指定键对应的成员
    auto i = obj.FindMember(key);
    // 如果找到指定键对应的成员，并且其值是 unsigned 类型，则返回该值
    if (i != obj.MemberEnd() && i->value.IsUint()) {
        return i->value.GetUint();
    }

    // 返回默认值
    return defaultValue;
}


// 将 double 类型的值转换为 rapidjson::Value 对象，如果值不是正常数则根据 zero 参数返回相应的值
rapidjson::Value xmrig::Json::normalize(double value, bool zero)
{
    using namespace rapidjson;

    // 如果值不是正常数，则根据 zero 参数返回相应的值
    if (!std::isnormal(value)) {
        return zero ? Value(0.0) : Value(kNullType);
    }

    // 返回经过处理的值
    return Value(floor(value * 100.0) / 100.0);
}


// 从输入流中读取数据，根据偏移量获取指定位置的行和字符位置，并存储到相应的变量中
bool xmrig::Json::convertOffset(std::istream &ifs, size_t offset, size_t &line, size_t &pos, std::vector<std::string> &s)
{
    std::string prev_t;
    std::string t;
    line = 0;
    pos = 0;
    size_t k = 0;

    // 循环读取输入流中的数据
    while (!ifs.eof()) {
        prev_t = t;
        std::getline(ifs, t);
        k += t.length() + 1;
        ++line;

        // 如果偏移量超过当前位置，则计算字符位置并存储相应的行数据
        if (k > offset) {
            pos = offset + t.length() + 1 - k + 1;

            s.clear();
            if (!prev_t.empty()) {
                s.emplace_back(prev_t);
            }
            s.emplace_back(t);

            return true;
        }
    }

    // 如果偏移量超过文件末尾，则返回 false
    return false;
}


// JsonReader 类的构造函数，初始化 m_obj 为 kNullValue
xmrig::JsonReader::JsonReader() :
    m_obj(kNullValue)
{}


// 判断 m_obj 是否为空
bool xmrig::JsonReader::isEmpty() const
{
    return Json::isEmpty(m_obj);
}
```