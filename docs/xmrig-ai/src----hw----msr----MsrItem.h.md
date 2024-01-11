# `xmrig\src\hw\msr\MsrItem.h`

```
/* XMRig
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以自由地重新分发和/或修改
 *   根据 GNU 通用公共许可证的条款，发布
 *   版本 3 或 (根据您的选择) 任何更高版本。
 *
 *   本程序是基于希望它有用的目的分发的，
 *   但没有任何担保；甚至没有暗示的担保
 *   适销性或特定用途的适用性。详细了解
 *   GNU 通用公共许可证的更多详细信息。
 *
 *   如果没有收到 GNU 通用公共许可证的副本
 *   请参阅 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_MSRITEM_H
#define XMRIG_MSRITEM_H


#include "base/tools/String.h"


#include <limits>
#include <vector>


namespace xmrig
{


class MsrItem
{
public:
    // 定义一个常量，表示没有掩码
    constexpr static uint64_t kNoMask = std::numeric_limits<uint64_t>::max();

    // 默认构造函数
    inline MsrItem() = default;
    // 构造函数，接受寄存器、值和掩码作为参数
    inline MsrItem(uint32_t reg, uint64_t value, uint64_t mask = kNoMask) : m_reg(reg), m_value(value), m_mask(mask) {}

    // 从 JSON 值构造 MsrItem 对象
    MsrItem(const rapidjson::Value &value);

    // 判断 MsrItem 对象是否有效
    inline bool isValid() const     { return m_reg > 0; }
    // 获取寄存器
    inline uint32_t reg() const     { return m_reg; }
    // 获取值
    inline uint64_t value() const   { return m_value; }
    // 获取掩码
    inline uint64_t mask() const    { return m_mask; }

    // 根据旧值、新值和掩码计算新的值
    static inline uint64_t maskedValue(uint64_t old_value, uint64_t new_value, uint64_t mask)
    {
        return (new_value & mask) | (old_value & ~mask);
    }

    // 将 MsrItem 对象转换为 JSON 值
    rapidjson::Value toJSON(rapidjson::Document &doc) const;
    // 将 MsrItem 对象转换为字符串
    String toString() const;

private:
    uint32_t m_reg      = 0;
    uint64_t m_value    = 0;
    uint64_t m_mask     = kNoMask;
};


// 定义 MsrItem 的向量类型
using MsrItems = std::vector<MsrItem>;


} /* namespace xmrig */


#endif /* XMRIG_MSRITEM_H */
```