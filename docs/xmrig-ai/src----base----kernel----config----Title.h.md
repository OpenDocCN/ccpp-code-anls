# `xmrig\src\base\kernel\config\Title.h`

```cpp
/*
 * XMRig
 * 版权所有 (c) 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的 GNU 通用公共许可证的条款
 * 无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用的分发
 * 但没有任何保证；甚至没有对适销性或特定用途的隐含保证。请参阅
 * GNU 通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_TITLE_H
#define XMRIG_TITLE_H

// 包含前向声明的 rapidjson 头文件
#include "3rdparty/rapidjson/fwd.h"
// 包含 String 类的头文件
#include "base/tools/String.h"

// 命名空间 xmrig
namespace xmrig {

// 类 Title
class Title
{
public:
    // 默认构造函数
    Title() = default;
    // 从 rapidjson::Value 对象构造 Title 对象
    Title(const rapidjson::Value &value);

    // 返回是否启用
    inline bool isEnabled() const   { return m_enabled; }

    // 转换为 JSON 对象
    rapidjson::Value toJSON() const;
    // 返回值
    String value() const;

private:
    // 是否启用
    bool m_enabled  = true;
    // 值
    String m_value;
};

} // 命名空间 xmrig

#endif /* XMRIG_TITLE_H */
```