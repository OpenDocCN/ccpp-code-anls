# `xmrig\src\hw\dmi\DmiMemory.h`

```
/* XMRig
 * 版权所有 (c) 2000-2002 Alan Cox     <alan@redhat.com>
 * 版权所有 (c) 2005-2020 Jean Delvare <jdelvare@suse.de>
 * 版权所有 (c) 2018-2021 SChernykh    <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig        <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它，遵循 GNU 通用公共许可证的条款，由自由软件基金会发布，可以选择遵循许可证的第 3 版或者（自行选择）之后的版本。
 *
 * 本程序是基于希望它有用的前提下分发的，但没有任何担保；甚至没有适用于特定目的的隐含担保。更多详情请参见 GNU 通用公共许可证。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_DMIMEMORY_H
#define XMRIG_DMIMEMORY_H


#include "base/tools/String.h"


namespace xmrig {


struct dmi_header;


class DmiMemory
{
public:
    DmiMemory() = default;
    DmiMemory(dmi_header *h);

    // 检查内存信息是否有效
    inline bool isValid() const             { return !m_slot.isEmpty(); }
    // 返回内存银行信息
    inline const String &bank() const       { return m_bank; }
    // 返回内存ID信息
    inline const String &id() const         { return m_id.isNull() ? m_slot : m_id; }
    // 返回内存产品信息
    inline const String &product() const    { return m_product; }
    // 返回内存插槽信息
    inline const String &slot() const       { return m_slot; }
    // 返回内存供应商信息
    inline const String &vendor() const     { return m_vendor; }
    // 返回内存总线宽度
    inline uint16_t totalWidth() const      { return m_totalWidth; }
    // 返回内存电压
    inline uint16_t voltage() const         { return m_voltage; }
    // 返回内存数据总线宽度
    inline uint16_t width() const           { return m_width; }
    // 返回内存大小
    inline uint64_t size() const            { return m_size; }
    // 返回内存速度
    inline uint64_t speed() const           { return m_speed; }
    // 返回内存等级
    inline uint8_t rank() const             { return m_rank; }
    // 返回一个指向常量字符的指针，表示设备的形式因素
    const char *formFactor() const;
    
    // 返回一个指向常量字符的指针，表示设备的类型
    const char *type() const;
#ifdef XMRIG_FEATURE_API
    // 如果定义了 XMRIG_FEATURE_API，则声明一个函数toJSON，将rapidjson::Document转换为rapidjson::Value
    rapidjson::Value toJSON(rapidjson::Document &doc) const;
#endif

private:
    // 声明一个私有函数setId，用于设置slot和bank的数值
    void setId(const char *slot, const char *bank);

    // 声明一系列私有成员变量，包括m_bank、m_id、m_product、m_slot、m_vendor等
    String m_bank;
    String m_id;
    String m_product;
    String m_slot;
    String m_vendor;
    uint16_t m_totalWidth   = 0; // 初始化m_totalWidth为0
    uint16_t m_voltage      = 0; // 初始化m_voltage为0
    uint16_t m_width        = 0; // 初始化m_width为0
    uint64_t m_size         = 0; // 初始化m_size为0
    uint64_t m_speed        = 0; // 初始化m_speed为0
    uint8_t m_formFactor    = 0; // 初始化m_formFactor为0
    uint8_t m_rank          = 0; // 初始化m_rank为0
    uint8_t m_type          = 0; // 初始化m_type为0
};


} /* namespace xmrig */
// 结束xmrig命名空间

#endif /* XMRIG_DMIMEMORY_H */
// 结束条件编译指令#endif XMRIG_DMIMEMORY_H
```