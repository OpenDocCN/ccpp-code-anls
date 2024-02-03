# `xmrig\src\hw\dmi\DmiMemory.cpp`

```cpp
/* XMRig
 * 版权所有 (c) 2000-2002 Alan Cox     <alan@redhat.com>
 * 版权所有 (c) 2005-2020 Jean Delvare <jdelvare@suse.de>
 * 版权所有 (c) 2018-2021 SChernykh    <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig        <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以自由地重新发布和/或修改
 *   根据 GNU 通用公共许可证的条款，发布的版本为
 *   (根据您的选择) 任何后续版本。
 *
 *   本程序是基于希望它有用的目的分发的，
 *   但没有任何担保；甚至没有适销性或特定用途的隐含担保。
 *   有关更多详细信息，请参阅 GNU 通用公共许可证。
 *
 *   如果没有收到 GNU 通用公共许可证的副本
 *   请参阅 <http://www.gnu.org/licenses/>。
 */


#include "hw/dmi/DmiMemory.h"
#include "3rdparty/fmt/format.h"
#include "3rdparty/rapidjson/document.h"
#include "hw/dmi/DmiTools.h"


#include <algorithm>
#include <array>
#include <regex>


namespace xmrig {


static const char *kIdFormat = "DIMM_{}{}";


static inline uint16_t dmi_memory_device_width(uint16_t code)
{
    // 如果代码为0xFFFF或0，则返回0，否则返回代码本身
    return (code == 0xFFFF || code == 0) ? 0 : code;
}


static const char *dmi_memory_device_form_factor(uint8_t code)
{
    // 存储内存设备的尺寸因子的字符串数组
    static const std::array<const char *, 0x10> form_factor
    {
        "Other",
        "Unknown",
        "SIMM",
        "SIP",
        "Chip",
        "DIP",
        "ZIP",
        "Proprietary Card",
        "DIMM",
        "TSOP",
        "Row Of Chips",
        "RIMM",
        "SODIMM",
        "SRIMM",
        "FB-DIMM",
        "Die"
    };

    // 如果代码在有效范围内，则返回对应的尺寸因子字符串
    if (code >= 0x01 && code <= form_factor.size()) {
        return form_factor[code - 0x01];
    }

    // 否则返回"Unknown"
    return form_factor[1];
}


static const char *dmi_memory_device_type(uint8_t code)
{
    # 定义一个静态常量数组，包含0x23个字符串，表示不同类型的内存
    static const std::array<const char *, 0x23> type
    {
        "Other", /* 0x01 */
        "Unknown",
        "DRAM",
        "EDRAM",
        "VRAM",
        "SRAM",
        "RAM",
        "ROM",
        "Flash",
        "EEPROM",
        "FEPROM",
        "EPROM",
        "CDRAM",
        "3DRAM",
        "SDRAM",
        "SGRAM",
        "RDRAM",
        "DDR",
        "DDR2",
        "DDR2 FB-DIMM",
        "Reserved",
        "Reserved",
        "Reserved",
        "DDR3",
        "FBD2",
        "DDR4",
        "LPDDR",
        "LPDDR2",
        "LPDDR3",
        "LPDDR4",
        "Logical non-volatile device",
        "HBM",
        "HBM2",
        "DDR5",
        "LPDDR5"
    };
    
    # 如果给定的code在0x01到type数组的大小之间
    if (code >= 0x01 && code <= type.size()) {
        # 返回type数组中对应code的元素
        return type[code - 0x01];
    }
    
    # 如果code不在0x01到type数组的大小之间，则返回type数组中索引为1的元素
    return type[1];
}

// 返回内存设备速度，如果 code1 为 0xFFFF，则返回 code2，否则返回 code1
static uint64_t dmi_memory_device_speed(uint16_t code1, uint32_t code2)
{
    return (code1 == 0xFFFF) ? code2 : code1;
}

} // namespace xmrig

// 构造函数，初始化内存设备信息
xmrig::DmiMemory::DmiMemory(dmi_header *h)
{
    // 如果长度小于 0x15，则直接返回
    if (h->length < 0x15) {
        return;
    }

    // 获取总线宽度和数据宽度
    m_totalWidth = dmi_memory_device_width(dmi_get<uint16_t>(h, 0x08));
    m_width      = dmi_memory_device_width(dmi_get<uint16_t>(h, 0x0A));

    // 获取内存大小
    auto size = dmi_get<uint16_t>(h, 0x0C);
    if (h->length >= 0x20 && size == 0x7FFF) {
        m_size = (dmi_get<uint32_t>(h, 0x1C) & 0x7FFFFFFFUL) * 1024ULL * 1024ULL;
    }
    else if (size) {
        m_size = (1024ULL * (size & 0x7FFF) * ((size & 0x8000) ? 1 : 1024ULL));
    }

    // 设置 ID
    setId(dmi_string(h, 0x10), dmi_string(h, 0x11));

    // 设置内存设备的形状和类型
    m_formFactor = h->data[0x0E];
    m_type       = h->data[0x12];

    // 如果大小为 0 或长度小于 0x17，则直接返回
    if (!m_size || h->length < 0x17) {
        return;
    }

    // 获取内存速度
    m_speed = dmi_memory_device_speed(dmi_get<uint16_t>(h, 0x15), h->length >= 0x5C ? dmi_get<uint32_t>(h, 0x54) : 0) * 1000000ULL;

    // 如果长度小于 0x1B，则直接返回
    if (h->length < 0x1B) {
        return;
    }

    // 获取制造商和产品信息
    m_vendor  = dmi_string(h, 0x17);
    m_product = dmi_string(h, 0x1A);

    // 如果长度小于 0x1C，则直接返回
    if (h->length < 0x1C) {
        return;
    }

    // 获取内存等级
    m_rank = h->data[0x1B] & 0x0F;

    // 如果长度小于 0x22，则直接返回
    if (h->length < 0x22) {
        return;
    }

    // 获取配置速度
    const uint64_t configuredSpeed = dmi_memory_device_speed(dmi_get<uint16_t>(h, 0x20), h->length >= 0x5C ? dmi_get<uint32_t>(h, 0x58) : 0) * 1000000ULL;
    m_speed = configuredSpeed ? configuredSpeed : m_speed;

    // 如果长度小于 0x28，则直接返回
    if (h->length < 0x28) {
        return;
    }

    // 获取电压
    m_voltage = dmi_get<uint16_t>(h, 0x26);
}

// 返回内存设备形状
const char *xmrig::DmiMemory::formFactor() const
{
    return dmi_memory_device_form_factor(m_formFactor);
}

// 返回内存设备类型
const char *xmrig::DmiMemory::type() const
{
    return dmi_memory_device_type(m_type);
}

#ifdef XMRIG_FEATURE_API
// 转换为 JSON 格式
rapidjson::Value xmrig::DmiMemory::toJSON(rapidjson::Document &doc) const
{
    using namespace rapidjson;

    auto &allocator = doc.GetAllocator();
    Value out(kObjectType);
    # 添加成员 "id"，值为当前对象的唯一标识符的 JSON 表示，使用给定的分配器
    out.AddMember("id",             id().toJSON(doc), allocator);
    # 添加成员 "slot"，值为当前对象的插槽的 JSON 表示，使用给定的分配器
    out.AddMember("slot",           m_slot.toJSON(doc), allocator);
    # 添加成员 "type"，值为当前对象的类型的字符串表示，使用给定的分配器
    out.AddMember("type",           StringRef(type()), allocator);
    # 添加成员 "form_factor"，值为当前对象的形状因子的字符串表示，使用给定的分配器
    out.AddMember("form_factor",    StringRef(formFactor()), allocator);
    # 添加成员 "size"，值为当前对象的大小，使用给定的分配器
    out.AddMember("size",           m_size, allocator);
    # 添加成员 "speed"，值为当前对象的速度，使用给定的分配器
    out.AddMember("speed",          m_speed, allocator);
    # 添加成员 "rank"，值为当前对象的等级，使用给定的分配器
    out.AddMember("rank",           m_rank, allocator);
    # 添加成员 "voltage"，值为当前对象的电压，使用给定的分配器
    out.AddMember("voltage",        m_voltage, allocator);
    # 添加成员 "width"，值为当前对象的宽度，使用给定的分配器
    out.AddMember("width",          m_width, allocator);
    # 添加成员 "total_width"，值为当前对象的总宽度，使用给定的分配器
    out.AddMember("total_width",    m_totalWidth, allocator);
    # 添加成员 "vendor"，值为当前对象的供应商的 JSON 表示，使用给定的分配器
    out.AddMember("vendor",         m_vendor.toJSON(doc), allocator);
    # 添加成员 "product"，值为当前对象的产品的 JSON 表示，使用给定的分配器
    out.AddMember("product",        m_product.toJSON(doc), allocator);
    # 添加成员 "bank"，值为当前对象的银行的 JSON 表示，使用给定的分配器
    out.AddMember("bank",           m_bank.toJSON(doc), allocator);
    
    # 返回 JSON 对象
    return out;
#endif


void xmrig::DmiMemory::setId(const char *slot, const char *bank)
{
    // 设置内存槽位和内存条所在的银行
    m_slot = slot;
    m_bank = bank;

    // 如果内存槽位或银行为空，则直接返回
    if (!slot || !bank) {
        return;
    }

    try {
        std::cmatch cm;
        // 如果内存槽位符合正则表达式"^Channel([A-Z])[-_]DIMM(\\d+)$"，则生成对应的ID
        if (std::regex_match(slot, cm, std::regex("^Channel([A-Z])[-_]DIMM(\\d+)$", std::regex_constants::icase))) {
            m_id = fmt::format(kIdFormat, cm.str(1), cm.str(2)).c_str();
        }
        // 如果银行符合正则表达式"CHANNEL ([A-Z])$"，则继续判断内存槽位是否符合正则表达式"^DIMM (\\d+)$"，如果符合则生成对应的ID
        else if (std::regex_search(bank, cm, std::regex("CHANNEL ([A-Z])$"))) {
            std::cmatch cm2;
            if (std::regex_match(slot, cm2, std::regex("^DIMM (\\d+)$"))) {
                m_id = fmt::format(kIdFormat, cm.str(1), cm2.str(1)).c_str();
            }
        }
    } catch (...) {}
}
```