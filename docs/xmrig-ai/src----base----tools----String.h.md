# `xmrig\src\base\tools\String.h`

```cpp
/**
 * @brief XMRig字符串类的头文件
 *
 * 包含了版权声明和GNU通用公共许可证的条款
 */
#ifndef XMRIG_STRING_H
#define XMRIG_STRING_H

/**
 * @brief 引入rapidjson的前置声明
 */
#include "3rdparty/rapidjson/fwd.h"

/**
 * @brief 引入必要的头文件
 */
#include <utility>
#include <vector>

/**
 * @brief XMRig命名空间
 */
namespace xmrig {

/**
 * @brief 简单的C字符串包装器
 *
 * 1. 我知道std:string。
 * 2. 由于某些原因，我更喜欢在矿工中不使用std:string，例如因为MSYS2构建的文件大小。
 * 3. 支持nullptr和JSON转换。
 */
class String
{
public:
    /**
     * @brief 默认构造函数
     */
    inline String() = default;
    /**
     * @brief 使用C字符串构造String对象
     */
    inline String(char *str) : m_data(str), m_size(str == nullptr ? 0 : strlen(str))    {}
    /**
     * @brief 移动构造函数
     */
    inline String(String &&other) noexcept : m_data(other.m_data), m_size(other.m_size) { other.m_data = nullptr; other.m_size = 0; }

    /**
     * @brief 使用指定大小的C字符串构造String对象
     */
    String(const char *str, size_t size);
    /**
     * @brief 使用C字符串构造String对象
     */
    String(const char *str);
    /**
     * @brief 使用rapidjson::Value对象构造String对象
     */
    String(const rapidjson::Value &value);
    /**
     * @brief 拷贝构造函数
     */
    String(const String &other);

    /**
     * @brief 析构函数
     */
    inline ~String() { delete [] m_data; }

    /**
     * @brief 判断是否与指定C字符串相等
     */
    bool isEqual(const char *str) const;
    /**
     * @brief 判断是否与另一个String对象相等
     */
    bool isEqual(const String &other) const;

    /**
     * @brief 判断是否包含指定的C字符串
     */
    inline bool contains(const char *str) const { return isNull() ? false : strstr(m_data, str) != nullptr; }

    /**
     * @brief 判断是否为空
     */
    inline bool isEmpty() const          { return size() == 0; }
    // 检查字符串是否为空，即数据指针是否为空
    inline bool isNull() const           { return m_data == nullptr; }
    // 检查字符串是否有效，即数据指针是否不为空
    inline bool isValid() const          { return m_data != nullptr; }
    // 返回字符串数据的指针
    inline char *data()                  { return m_data; }
    // 返回字符串数据的常量指针
    inline const char *data() const      { return m_data; }
    // 返回字符串数据的大小
    inline size_t size() const           { return m_size; }

    // 重载不等于操作符，判断字符串是否不相等
    inline bool operator!=(const char *str) const      { return !isEqual(str); }
    // 重载不等于操作符，判断字符串是否不相等
    inline bool operator!=(const String &other) const  { return !isEqual(other); }
    // 重载小于操作符，判断字符串的字典序是否小于另一个字符串
    inline bool operator<(const String &str) const     { return !isEmpty() && !str.isEmpty() && strcmp(data(), str.data()) < 0; }
    // 重载等于操作符，判断字符串是否相等
    inline bool operator==(const char *str) const      { return isEqual(str); }
    // 重载等于操作符，判断字符串是否相等
    inline bool operator==(const String &other) const  { return isEqual(other); }
    // 转换为常量字符指针
    inline operator const char*() const                { return m_data; }
    // 赋值操作符，将传入的字符指针移动到当前字符串对象
    inline String &operator=(char *str)                { move(str); return *this; }
    // 赋值操作符，将传入的常量字符指针复制到当前字符串对象
    inline String &operator=(const char *str)          { copy(str); return *this; }
    // 赋值操作符，将传入的字符串对象复制到当前字符串对象
    inline String &operator=(const String &str)        { copy(str); return *this; }
    // 赋值操作符，将空指针赋值给当前字符串对象
    inline String &operator=(std::nullptr_t)           { delete [] m_data; m_data = nullptr; m_size = 0; return *this; }
    // 移动赋值操作符，将传入的字符串对象移动到当前字符串对象
    inline String &operator=(String &&other) noexcept  { move(std::move(other)); return *this; }

    // 转换为 JSON 对象
    rapidjson::Value toJSON() const;
    // 转换为 JSON 对象
    rapidjson::Value toJSON(rapidjson::Document &doc) const;
    // 根据分隔符分割字符串，返回分割后的字符串数组
    std::vector<String> split(char sep) const;
    // 将字符串转换为小写
    String &toLower();
    // 将字符串转换为大写
    String &toUpper();

    // 将字符串数组按照指定分隔符连接成一个字符串
    static String join(const std::vector<String> &vec, char sep);
# 声明私有成员函数，用于复制字符串和移动字符串
private:
    void copy(const char *str);
    void copy(const String &other);
    void move(char *str);
    void move(String &&other);

    # 声明私有成员变量，用于存储字符串数据和大小
    char *m_data    = nullptr;
    size_t m_size   = 0;
};


# 结束命名空间声明
} /* namespace xmrig */


# 结束头文件声明
#endif /* XMRIG_STRING_H */
```