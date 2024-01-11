# `xmrig\src\base\io\json\Json.h`

```
/*
 * XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款，
 * 无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。有关更多详细信息，请参见
 * GNU通用公共许可证。
 *
 * 您应该已经收到GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */
 
#ifndef XMRIG_JSON_H
#define XMRIG_JSON_H

#include "base/kernel/interfaces/IJsonReader.h"  // 包含IJsonReader接口的头文件

#include <string>  // 包含string头文件

namespace xmrig {

class Json
{
public:
    static bool getBool(const rapidjson::Value &obj, const char *key, bool defaultValue = false);  // 获取布尔值的静态方法
    static bool isEmpty(const rapidjson::Value &obj);  // 判断rapidjson::Value对象是否为空的静态方法
    static const char *getString(const rapidjson::Value &obj, const char *key, const char *defaultValue = nullptr);  // 获取字符串的静态方法
    static const rapidjson::Value &getArray(const rapidjson::Value &obj, const char *key);  // 获取数组的静态方法
    static const rapidjson::Value &getObject(const rapidjson::Value &obj, const char *key);  // 获取对象的静态方法
    static const rapidjson::Value &getValue(const rapidjson::Value &obj, const char *key);  // 获取值的静态方法
    static double getDouble(const rapidjson::Value &obj, const char *key, double defaultValue = 0);  // 获取双精度浮点数的静态方法
    static int getInt(const rapidjson::Value &obj, const char *key, int defaultValue = 0);  // 获取整数的静态方法
    static int64_t getInt64(const rapidjson::Value &obj, const char *key, int64_t defaultValue = 0);  // 获取64位整数的静态方法
    static String getString(const rapidjson::Value &obj, const char *key, size_t maxSize);  // 获取字符串的静态方法
    # 从 rapidjson::Value 对象中获取指定键的 uint64_t 类型数值，如果键不存在则返回默认值
    static uint64_t getUint64(const rapidjson::Value &obj, const char *key, uint64_t defaultValue = 0);
    # 从 rapidjson::Value 对象中获取指定键的 unsigned 类型数值，如果键不存在则返回默认值
    static unsigned getUint(const rapidjson::Value &obj, const char *key, unsigned defaultValue = 0);
    
    # 从指定文件名读取 JSON 文档内容到 rapidjson::Document 对象中
    static bool get(const char *fileName, rapidjson::Document &doc);
    # 将 rapidjson::Document 对象保存为 JSON 格式到指定文件中
    static bool save(const char *fileName, const rapidjson::Document &doc);
    
    # 将指定文件中的偏移量转换为行号、位置和字符串内容，存储到相应的参数中
    static bool convertOffset(const char *fileName, size_t offset, size_t &line, size_t &pos, std::vector<std::string> &s);
    # 将 double 类型的数值转换为 rapidjson::Value 对象，如果为零则进行规范化处理
    static rapidjson::Value normalize(double value, bool zero);
// JsonReader 类的私有成员函数，用于将文件流转换为指定偏移位置的行号、列号和字符串向量
private:
    static bool convertOffset(std::istream &ifs, size_t offset, size_t &line, size_t &pos, std::vector<std::string> &s);
};


// JsonReader 类的公有成员函数，用于从 JSON 对象中获取不同类型的值
class JsonReader : public IJsonReader
{
public:
    // 默认构造函数
    JsonReader();
    // 带参构造函数，使用给定的 rapidjson::Value 对象初始化
    inline JsonReader(const rapidjson::Value &obj) : m_obj(obj) {}

    // 从 JSON 对象中获取布尔值，如果键不存在则返回默认值
    inline bool getBool(const char *key, bool defaultValue = false) const override                   { return Json::getBool(m_obj, key, defaultValue); }
    // 从 JSON 对象中获取字符串，如果键不存在则返回默认值
    inline const char *getString(const char *key, const char *defaultValue = nullptr) const override { return Json::getString(m_obj, key, defaultValue); }
    // 从 JSON 对象中获取数组
    inline const rapidjson::Value &getArray(const char *key) const override                          { return Json::getArray(m_obj, key); }
    // 从 JSON 对象中获取子对象
    inline const rapidjson::Value &getObject(const char *key) const override                         { return Json::getObject(m_obj, key); }
    // 从 JSON 对象中获取值
    inline const rapidjson::Value &getValue(const char *key) const override                          { return Json::getValue(m_obj, key); }
    // 返回 JSON 对象本身
    inline const rapidjson::Value &object() const override                                           { return m_obj; }
    // 从 JSON 对象中获取双精度浮点数，如果键不存在则返回默认值
    inline double getDouble(const char *key, double defaultValue = 0) const override                 { return Json::getDouble(m_obj, key, defaultValue); }
    // 从 JSON 对象中获取整数，如果键不存在则返回默认值
    inline int getInt(const char *key, int defaultValue = 0) const override                          { return Json::getInt(m_obj, key, defaultValue); }
    // 从 JSON 对象中获取 64 位整数，如果键不存在则返回默认值
    inline int64_t getInt64(const char *key, int64_t defaultValue = 0) const override                { return Json::getInt64(m_obj, key, defaultValue); }
    // 从 JSON 对象中获取指定长度的字符串，如果键不存在则返回默认值
    inline String getString(const char *key, size_t maxSize) const override                          { return Json::getString(m_obj, key, maxSize); }
    // 从 JSON 对象中获取 64 位无符号整数，如果键不存在则返回默认值
    inline uint64_t getUint64(const char *key, uint64_t defaultValue = 0) const override             { return Json::getUint64(m_obj, key, defaultValue); }
}
    # 重写父类的 getUint 方法，根据给定的键获取对应的无符号整数值，如果键不存在则返回默认值
    inline unsigned getUint(const char *key, unsigned defaultValue = 0) const override               { return Json::getUint(m_obj, key, defaultValue); }

    # 重写父类的 isEmpty 方法，检查当前对象是否为空
    bool isEmpty() const override;
# 声明私有成员变量 m_obj，类型为 rapidjson::Value 的引用
private:
    const rapidjson::Value &m_obj;
};  /* namespace xmrig */
#endif /* XMRIG_JSON_H */
```