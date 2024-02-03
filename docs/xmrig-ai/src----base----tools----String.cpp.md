# `xmrig\src\base\tools\String.cpp`

```cpp
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款，
 * 无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用的分发，
 * 但没有任何保证；甚至没有对适销性或特定用途的隐含保证。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "base/tools/String.h"
#include "3rdparty/rapidjson/document.h"


#include <cctype>


xmrig::String::String(const char *str, size_t size) :
    m_size(size)
{
    if (str == nullptr) {
        m_size = 0;

        return;
    }

    m_data = new char[m_size + 1];
    memcpy(m_data, str, m_size);
    m_data[m_size] = '\0';
}


xmrig::String::String(const char *str) :
    m_size(str == nullptr ? 0 : strlen(str))
{
    if (str == nullptr) {
        return;
    }

    m_data = new char[m_size + 1];
    memcpy(m_data, str, m_size + 1);
}


xmrig::String::String(const rapidjson::Value &value)
{
    if (!value.IsString()) {
        return;
    }

    if ((m_size = value.GetStringLength()) == 0) {
        return;
    }

    m_data = new char[m_size + 1];
    memcpy(m_data, value.GetString(), m_size);
    m_data[m_size] = '\0';
}


xmrig::String::String(const String &other) :
    m_size(other.m_size)
{
    if (other.m_data == nullptr) {
        return;
    }

    m_data = new char[m_size + 1];
    memcpy(m_data, other.m_data, m_size + 1);
}


bool xmrig::String::isEqual(const char *str) const
    # 返回一个布尔值，判断条件为：
    # m_data 不为空且 str 不为空，且 m_data 和 str 相等
    # 或者 m_data 为空且 str 为空
    return (m_data != nullptr && str != nullptr && strcmp(m_data, str) == 0) || (m_data == nullptr && str == nullptr);
// 检查两个字符串是否相等，如果长度不同则直接返回false，否则比较数据内容是否相同
bool xmrig::String::isEqual(const String &other) const
{
    if (m_size != other.m_size) {
        return false;
    }

    return (m_data != nullptr && other.m_data != nullptr && memcmp(m_data, other.m_data, m_size) == 0) || (m_data == nullptr && other.m_data == nullptr);
}

// 将字符串转换为 JSON 格式的数值，如果字符串为空则返回空值，否则返回字符串内容
rapidjson::Value xmrig::String::toJSON() const
{
    using namespace rapidjson;

    return isNull() ? Value(kNullType) : Value(StringRef(m_data));
}

// 将字符串转换为 JSON 格式的数值，如果字符串为空则返回空值，否则返回字符串内容，并使用给定的内存分配器
rapidjson::Value xmrig::String::toJSON(rapidjson::Document &doc) const
{
    using namespace rapidjson;

    return isNull() ? Value(kNullType) : Value(m_data, doc.GetAllocator());
}

// 根据指定分隔符将字符串分割成多个子字符串，并返回子字符串的向量
std::vector<xmrig::String> xmrig::String::split(char sep) const
{
    // ...
}

// 将字符串中的字母转换为小写形式
xmrig::String &xmrig::String::toLower()
{
    // ...
}

// 将字符串中的字母转换为大写形式
xmrig::String &xmrig::String::toUpper()
{
    // ...
}

// 将给定的字符串向量使用指定的分隔符连接成一个新的字符串
xmrig::String xmrig::String::join(const std::vector<xmrig::String> &vec, char sep)
{
    // ...
}
    # 遍历字符串向量，将每个字符串的数据拷贝到缓冲区中
    for (const String &str : vec) {
        # 使用memcpy将字符串数据拷贝到缓冲区中
        memcpy(buf + offset, str.data(), str.size());
        
        # 更新偏移量，指向下一个字符串数据应该拷贝的位置
        offset += str.size() + 1;
        
        # 如果偏移量小于缓冲区大小，将分隔符写入缓冲区
        if (offset < size) {
            buf[offset - 1] = sep;
        }
    }

    # 在缓冲区的最后一个位置写入字符串结束符
    buf[size - 1] = '\0';

    # 将缓冲区中的数据转换为字符串并返回
    return String(buf);
# 复制传入的字符串到当前对象中
void xmrig::String::copy(const char *str)
{
    # 释放当前对象的数据
    delete [] m_data;

    # 如果传入的字符串为空，则设置当前对象的大小为0，数据为空，然后返回
    if (str == nullptr) {
        m_size = 0;
        m_data = nullptr;

        return;
    }

    # 获取传入字符串的长度
    m_size = strlen(str);
    # 为当前对象分配内存空间
    m_data = new char[m_size + 1];
    # 将传入字符串的数据复制到当前对象中
    memcpy(m_data, str, m_size + 1);
}


# 复制另一个String对象的数据到当前对象中
void xmrig::String::copy(const String &other)
{
    # 如果当前对象的大小大于0且与另一个对象的大小相同，则直接复制数据并返回
    if (m_size > 0 && m_size == other.m_size) {
        memcpy(m_data, other.m_data, m_size + 1);

        return;
    }

    # 释放当前对象的数据
    delete [] m_data;

    # 如果另一个对象的数据为空，则设置当前对象的大小为0，数据为空，然后返回
    if (other.m_data == nullptr) {
        m_size = 0;
        m_data = nullptr;

        return;
    }

    # 获取另一个对象的大小，并为当前对象分配内存空间
    m_size = other.m_size;
    m_data = new char[m_size + 1];
    # 将另一个对象的数据复制到当前对象中
    memcpy(m_data, other.m_data, m_size + 1);
}


# 移动传入的字符串到当前对象中
void xmrig::String::move(char *str)
{
    # 释放当前对象的数据
    delete [] m_data;

    # 如果传入的字符串为空，则设置当前对象的大小为0，数据为空
    m_size = str == nullptr ? 0 : strlen(str);
    # 直接将传入的字符串指针赋值给当前对象的数据指针
    m_data = str;
}


# 移动另一个String对象的数据到当前对象中
void xmrig::String::move(String &&other)
{
    # 释放当前对象的数据
    delete [] m_data;

    # 将另一个对象的数据指针和大小赋值给当前对象，然后将另一个对象的数据指针和大小置空
    m_data = other.m_data;
    m_size = other.m_size;
    other.m_data = nullptr;
    other.m_size = 0;
}
```