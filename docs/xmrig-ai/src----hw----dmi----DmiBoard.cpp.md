# `xmrig\src\hw\dmi\DmiBoard.cpp`

```cpp
/* XMRig
 * 版权所有 (c) 2000-2002 Alan Cox     <alan@redhat.com>
 * 版权所有 (c) 2005-2020 Jean Delvare <jdelvare@suse.de>
 * 版权所有 (c) 2018-2021 SChernykh    <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig        <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以自由地重新分发和/或修改
 *   根据 GNU 通用公共许可证的条款，版本 3 或
 *   （根据您的选择）任何更高版本。
 *
 *   本程序是基于希望它有用的目的分发的，
 *   但没有任何担保；甚至没有适销性或特定用途的隐含担保。详见
 *   GNU 通用公共许可证获取更多详细信息。
 *
 *   如果没有收到 GNU 通用公共许可证的副本，
 *   请参阅 <http://www.gnu.org/licenses/>。
 */


#include "hw/dmi/DmiBoard.h"
#include "3rdparty/rapidjson/document.h"
#include "hw/dmi/DmiTools.h"


// 解析 DMI 数据结构中的主板信息
void xmrig::DmiBoard::decode(dmi_header *h)
{
    // 如果 DMI 数据结构的长度小于 0x08，则返回
    if (h->length < 0x08) {
        return;
    }

    // 解析 DMI 数据结构中的厂商信息
    m_vendor  = dmi_string(h, 0x04);
    // 解析 DMI 数据结构中的产品信息
    m_product = dmi_string(h, 0x05);
}


#ifdef XMRIG_FEATURE_API
// 将主板信息转换为 JSON 格式
rapidjson::Value xmrig::DmiBoard::toJSON(rapidjson::Document &doc) const
{
    using namespace rapidjson;

    // 获取 JSON 文档的分配器
    auto &allocator = doc.GetAllocator();
    // 创建一个空的 JSON 对象
    Value out(kObjectType);
    // 将厂商信息添加到 JSON 对象中
    out.AddMember("vendor",     m_vendor.toJSON(doc), allocator);
    // 将产品信息添加到 JSON 对象中
    out.AddMember("product",    m_product.toJSON(doc), allocator);

    return out;
}
#endif
```