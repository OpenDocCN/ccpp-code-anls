# `xmrig\src\hw\dmi\DmiReader.cpp`

```cpp
/* XMRig
 * 版权所有 (c) 2000-2002 Alan Cox     <alan@redhat.com>
 * 版权所有 (c) 2005-2020 Jean Delvare <jdelvare@suse.de>
 * 版权所有 (c) 2018-2021 SChernykh    <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig        <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以自由地重新分发和/或修改
 *   根据GNU通用公共许可证的条款，发布的版本为
 *   (根据您的选择)任何更高版本。
 *
 *   本程序是基于希望它有用的目的分发的，
 *   但没有任何担保；甚至没有暗示的担保
 *   适销性或特定用途的保证。详细了解
 *   GNU通用公共许可证的更多细节。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   这个程序。如果没有，请参阅<http://www.gnu.org/licenses/>。
 */

#include "hw/dmi/DmiReader.h"
#include "3rdparty/fmt/core.h"
#include "3rdparty/rapidjson/document.h"
#include "hw/dmi/DmiTools.h"


namespace xmrig {


static void dmi_get_header(dmi_header *h, uint8_t *data)
{
    // 从数据中读取DMI头部信息
    h->type   = data[0];
    h->length = data[1];
    h->handle = dmi_get<uint16_t>(data + 2);
    h->data   = data;
}


} // namespace xmrig


#ifdef XMRIG_FEATURE_API
rapidjson::Value xmrig::DmiReader::toJSON(rapidjson::Document &doc) const
{
    rapidjson::Value obj;
    // 将DmiReader对象转换为JSON对象
    toJSON(obj, doc);

    return obj;
}


void xmrig::DmiReader::toJSON(rapidjson::Value &out, rapidjson::Document &doc) const
{
    using namespace rapidjson;

    auto &allocator = doc.GetAllocator();
    out.SetObject();

    Value memory(kArrayType);
    memory.Reserve(m_memory.size(), allocator);

    // 遍历m_memory列表，将每个元素转换为JSON对象并添加到memory数组中
    for (const auto &value : m_memory) {
        memory.PushBack(value.toJSON(doc), allocator);
    }
    # 将字符串格式化后的版本号添加到 JSON 对象中，键为"smbios"，值为格式化后的版本号
    out.AddMember("smbios",     Value(fmt::format("{}.{}.{}", m_version >> 16, m_version >> 8 & 0xff, m_version & 0xff).c_str(), allocator), allocator);
    # 将系统信息转换为 JSON 对象后添加到 JSON 对象中，键为"system"，值为系统信息的 JSON 对象
    out.AddMember("system",     m_system.toJSON(doc), allocator);
    # 将主板信息转换为 JSON 对象后添加到 JSON 对象中，键为"board"，值为主板信息的 JSON 对象
    out.AddMember("board",      m_board.toJSON(doc), allocator);
    # 将内存信息添加到 JSON 对象中，键为"memory"，值为内存信息
    out.AddMember("memory",     memory, allocator);
#endif


bool xmrig::DmiReader::decode(uint8_t *buf, const Cleanup &cleanup)
{
    // 调用另一个decode函数，并将返回值保存到rc变量中
    const bool rc = decode(buf);

    // 调用cleanup函数
    cleanup();

    // 返回rc变量的值
    return rc;
}


bool xmrig::DmiReader::decode(uint8_t *buf)
{
    // 如果buf为空指针，则返回false
    if (!buf) {
        return false;
    }

    // 将buf赋值给data变量
    uint8_t *data = buf;
    // 将i变量赋值为0
    int i         = 0;

    // 当data + 4小于等于buf + m_size时，执行循环
    while (data + 4 <= buf + m_size) {
        // 创建dmi_header类型的h对象，并调用dmi_get_header函数将data指向的数据解析到h对象中
        dmi_header h{};
        dmi_get_header(&h, data);

        // 如果h.length小于4或者h.type等于127，则跳出循环
        if (h.length < 4 || h.type == 127) {
            break;
        }
        // i自增1
        i++;

        // 将data + h.length赋值给next变量
        uint8_t *next = data + h.length;
        // 当next - buf + 1小于m_size并且next[0]不等于0或者next[1]不等于0时，执行循环
        while (static_cast<uint32_t>(next - buf + 1) < m_size && (next[0] != 0 || next[1] != 0)) {
            next++;
        }

#       ifdef XMRIG_OS_APPLE
        // 当next - buf + 1小于m_size并且next[0]等于0并且next[1]等于0时，执行循环
        while ((unsigned long)(next - buf + 1) < m_size && (next[0] == 0 && next[1] == 0))
#       endif
        next += 2;

        // 如果next - buf大于m_size，则跳出循环
        if (static_cast<uint32_t>(next - buf) > m_size) {
            break;
        }

        // 根据h.type的值执行相应的操作
        switch (h.type) {
        case 1:
            // 调用m_system对象的decode函数，并将h作为参数传入
            m_system.decode(&h);
            break;

        case 2:
            // 调用m_board对象的decode函数，并将h作为参数传入
            m_board.decode(&h);
            break;

        case 17:
            // 将h对象保存到m_memory容器中
            m_memory.emplace_back(&h);
            break;

        default:
            break;
        }

        // 将next赋值给data变量
        data = next;
    }

    // 返回true
    return true;
}
```