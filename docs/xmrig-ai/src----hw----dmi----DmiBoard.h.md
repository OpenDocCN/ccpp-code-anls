# `xmrig\src\hw\dmi\DmiBoard.h`

```cpp
/* XMRig
 * 版权所有 (c) 2000-2002 Alan Cox     <alan@redhat.com>
 * 版权所有 (c) 2005-2020 Jean Delvare <jdelvare@suse.de>
 * 版权所有 (c) 2018-2021 SChernykh    <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig        <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以自由地重新发布和/或修改
 *   根据 GNU 通用公共许可证的条款，版本 3 或
 *   (根据您的选择) 任何更高版本。
 *
 *   本程序是基于希望它有用的目的分发的，
 *   但没有任何担保；甚至没有暗示的担保
 *   适销性或特定用途的适用性。详细了解
 *   GNU 通用公共许可证的更多细节。
 *
 *   如果没有收到 GNU 通用公共许可证的副本
 *   请参阅 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_DMIBOARD_H
#define XMRIG_DMIBOARD_H


#include "base/tools/String.h"


namespace xmrig {


struct dmi_header;


class DmiBoard
{
public:
    DmiBoard() = default;

    // 返回产品名称
    inline const String &product() const    { return m_product; }
    // 返回供应商名称
    inline const String &vendor() const     { return m_vendor; }
    // 检查产品名称和供应商名称是否有效
    inline bool isValid() const             { return !m_product.isEmpty() && !m_vendor.isEmpty(); }

    // 解码 DMI 头部信息
    void decode(dmi_header *h);

#   ifdef XMRIG_FEATURE_API
    // 将 DmiBoard 对象转换为 JSON 格式
    rapidjson::Value toJSON(rapidjson::Document &doc) const;
#   endif

private:
    // 产品名称
    String m_product;
    // 供应商名称
    String m_vendor;
};


} /* namespace xmrig */


#endif /* XMRIG_DMIBOARD_H */
```