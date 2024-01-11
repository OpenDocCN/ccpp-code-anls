# `xmrig\src\hw\dmi\DmiTools.cpp`

```
/* XMRig
 * 版权所有 (c) 2000-2002 Alan Cox     <alan@redhat.com>
 * 版权所有 (c) 2005-2020 Jean Delvare <jdelvare@suse.de>
 * 版权所有 (c) 2018-2021 SChernykh    <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig        <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以自由地重新发布和/或修改
 *   根据 GNU 通用公共许可证的条款，发布的版本为
 *   3 或 (根据您的选择) 任何更高版本。
 *
 *   本程序是基于希望它有用的目的分发的，
 *   但没有任何担保；甚至没有适销性或特定用途的隐含担保。
 *   有关更多详细信息，请参阅 GNU 通用公共许可证。
 *
 *   如果没有收到 GNU 通用公共许可证的副本，
 *   请参阅 <http://www.gnu.org/licenses/>。
 */


#include "hw/dmi/DmiTools.h"


#include <cstring>


namespace xmrig {


/* 用点替换非 ASCII 字符 */
static void ascii_filter(char *bp, size_t len)
{
    for (size_t i = 0; i < len; i++) {
        if (bp[i] < 32 || bp[i] >= 127) {
            bp[i] = '.';
        }
    }
}


static char *_dmi_string(dmi_header *dm, uint8_t s, bool filter)
{
    char *bp = reinterpret_cast<char *>(dm->data);

    bp += dm->length;
    while (s > 1 && *bp)  {
        bp += strlen(bp);
        bp++;
        s--;
    }

    if (!*bp) {
        return nullptr;
    }

    if (filter) {
        ascii_filter(bp, strlen(bp));
    }

    return bp;
}


const char *dmi_string(dmi_header *dm, size_t offset)
{
    if (offset < 4) {
        return nullptr;
    }

    return _dmi_string(dm, dm->data[offset], true);
}


} // namespace xmrig


注释：
```