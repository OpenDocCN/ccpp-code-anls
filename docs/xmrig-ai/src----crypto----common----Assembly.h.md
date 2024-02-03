# `xmrig\src\crypto\common\Assembly.h`

```cpp
/* XMRig
 * 版权所有（c）2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款，
 * 无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用的分发，
 * 但没有任何保证；甚至没有对适销性或特定用途的隐含保证。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_ASSEMBLY_H
#define XMRIG_ASSEMBLY_H


#include "3rdparty/rapidjson/fwd.h"


namespace xmrig {


class Assembly
{
public:
    enum Id : int {
        NONE,
        AUTO,
        INTEL,
        RYZEN,
        BULLDOZER,
        MAX
    };


    inline Assembly()                                                   {}  // 默认构造函数
    inline Assembly(Id id) : m_id(id)                                   {}  // 使用ID构造函数
    inline Assembly(const char *assembly) : m_id(parse(assembly))       {}  // 使用字符数组构造函数
    inline Assembly(const rapidjson::Value &value) : m_id(parse(value)) {}  // 使用rapidjson::Value构造函数

    static Id parse(const char *assembly, Id defaultValue = AUTO);  // 解析字符数组并返回ID
    static Id parse(const rapidjson::Value &value, Id defaultValue = AUTO);  // 解析rapidjson::Value并返回ID

    const char *toString() const;  // 返回ID对应的字符串
    rapidjson::Value toJSON() const;  // 返回ID对应的rapidjson::Value

    inline bool isEqual(const Assembly &other) const      { return m_id == other.m_id; }  // 检查两个Assembly对象的ID是否相等
    inline Id id() const                                  { return m_id; }  // 返回ID

    inline bool operator!=(Assembly::Id id) const         { return m_id != id; }  // 检查ID是否不相等
    inline bool operator!=(const Assembly &other) const   { return !isEqual(other); }  // 检查两个Assembly对象的ID是否不相等
    # 定义重载运算符，用于比较当前对象的 id 和给定的 id 是否相等
    inline bool operator==(Assembly::Id id) const         { return m_id == id; }
    # 定义重载运算符，用于比较当前对象和另一个 Assembly 对象是否相等
    inline bool operator==(const Assembly &other) const   { return isEqual(other); }
    # 定义类型转换运算符，将当前对象转换为 Assembly::Id 类型
    inline operator Assembly::Id() const                  { return m_id; }
private:
    // 声明私有成员变量 m_id，并初始化为 AUTO
    Id m_id = AUTO;
};


} /* namespace xmrig */


#endif /* XMRIG_ASSEMBLY_H */
// 结束命名空间 xmrig，并结束头文件的定义
```