# `xmrig\src\base\kernel\interfaces\IConsoleListener.h`

```
/*
 * XMRig
 * 版权所有 (c) 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的 GNU 通用公共许可证的条款
 * 无论是许可证的第 3 版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用的，但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。更多详情请参阅
 * GNU 通用公共许可证。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参阅 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_ICONSOLELISTENER_H
#define XMRIG_ICONSOLELISTENER_H


#include "base/tools/Object.h"


namespace xmrig {


class IConsoleListener
{
public:
    XMRIG_DISABLE_COPY_MOVE(IConsoleListener)

    IConsoleListener()          = default;
    virtual ~IConsoleListener() = default;

    virtual void onConsoleCommand(char command) = 0;
};


} /* namespace xmrig */


#endif // XMRIG_ICONSOLELISTENER_H
```