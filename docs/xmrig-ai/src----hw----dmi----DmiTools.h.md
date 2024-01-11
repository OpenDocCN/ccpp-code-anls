# `xmrig\src\hw\dmi\DmiTools.h`

```
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

#ifndef XMRIG_DMITOOLS_H
#define XMRIG_DMITOOLS_H


#include <cstddef>
#include <cstdint>
#include "base/tools/Alignment.h"


namespace xmrig {


struct dmi_header
{
    uint8_t type;       // DMI 数据类型
    uint8_t length;     // DMI 数据长度
    uint16_t handle;    // DMI 数据句柄
    uint8_t *data;      // DMI 数据指针
};


struct u64 {
    uint32_t l;         // 低位 32 位
    uint32_t h;         // 高位 32 位
};


template<typename T>
inline T dmi_get(const uint8_t *data)                   { return readUnaligned(reinterpret_cast<const T *>(data)); }

template<typename T>
inline T dmi_get(const dmi_header *h, size_t offset)    { return readUnaligned(reinterpret_cast<const T *>(h->data + offset)); }


const char *dmi_string(dmi_header *dm, size_t offset);


} /* namespace xmrig */


#endif /* XMRIG_DMITOOLS_H */
```