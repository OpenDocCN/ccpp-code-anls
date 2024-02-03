# `xmrig\src\hw\msr\MsrItem.cpp`

```cpp
/* XMRig
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以自由地重新发布和/或修改
 *   根据GNU通用公共许可证的条款，发布的版本为3或
 *   （根据您的选择）以后的版本。
 *
 *   本程序是基于希望它有用的目的分发的，
 *   但没有任何担保；甚至没有暗示的担保
 *   适销性或特定用途的适用性。详细信息请参见
 *   GNU通用公共许可证。
 *
 *   如果没有收到GNU通用公共许可证的副本
 *   请参阅<http://www.gnu.org/licenses/>。
 */


#include "hw/msr/MsrItem.h"
#include "3rdparty/rapidjson/document.h"


#include <cstdio>


// 根据 rapidjson::Value 对象构造 MsrItem 对象
xmrig::MsrItem::MsrItem(const rapidjson::Value &value)
{
    // 如果 value 不是字符串类型，则返回
    if (!value.IsString()) {
        return;
    }

    // 将字符串按照冒号分割成键值对
    auto kv = String(value.GetString()).split(':');
    // 如果键值对数量小于2，则返回
    if (kv.size() < 2) {
        return;
    }

    // 将字符串转换为无符号长整型，作为寄存器地址
    m_reg   = strtoul(kv[0], nullptr, 0);
    // 将字符串转换为无符号长长整型，作为寄存器值
    m_value = strtoull(kv[1], nullptr, 0);
    // 如果键值对数量大于2，则将第三个值转换为无符号长长整型，作为掩码值；否则使用默认值 kNoMask
    m_mask  = (kv.size() > 2) ? strtoull(kv[2], nullptr, 0) : kNoMask;
}


// 将 MsrItem 对象转换为 rapidjson::Value 对象
rapidjson::Value xmrig::MsrItem::toJSON(rapidjson::Document &doc) const
{
    // 调用 toString() 方法将 MsrItem 对象转换为 String 对象，再调用 toJSON() 方法将 String 对象转换为 rapidjson::Value 对象
    return toString().toJSON(doc);
}


// 将 MsrItem 对象转换为字符串
xmrig::String xmrig::MsrItem::toString() const
{
    // 定义字符串缓冲区的大小
    constexpr size_t size = 48;

    // 创建一个大小为 size 的字符数组，并初始化为零
    auto buf = new char[size]();

    // 如果掩码值不等于默认值 kNoMask，则将寄存器地址、寄存器值和掩码值格式化为字符串并存储在 buf 中
    if (m_mask != kNoMask) {
        snprintf(buf, size, "0x%" PRIx32 ":0x%" PRIx64 ":0x%" PRIx64, m_reg, m_value, m_mask);
    }
    // 否则，将寄存器地址和寄存器值格式化为字符串并存储在 buf 中
    else {
        snprintf(buf, size, "0x%" PRIx32 ":0x%" PRIx64, m_reg, m_value);
    }

    // 将 buf 转换为 String 对象并返回
    return buf;
}
```