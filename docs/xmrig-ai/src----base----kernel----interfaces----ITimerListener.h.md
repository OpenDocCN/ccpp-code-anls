# `xmrig\src\base\kernel\interfaces\ITimerListener.h`

```
/*
 * XMRig
 * 版权所有 (c) 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的 GNU 通用公共许可证的条款
 * 其中包括许可证的第3版或
 * （根据您的选择）任何以后的版本。
 *
 * 本程序是希望它有用的
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。详细信息请参见
 * GNU 通用公共许可证。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_ITIMERLISTENER_H
#define XMRIG_ITIMERLISTENER_H


#include "base/tools/Object.h"


namespace xmrig {


class Timer;


class ITimerListener
{
public:
    XMRIG_DISABLE_COPY_MOVE(ITimerListener)

    ITimerListener()            = default;
    virtual ~ITimerListener()   = default;

    virtual void onTimer(const Timer *timer) = 0;
};


} /* namespace xmrig */


#endif // XMRIG_ITIMERLISTENER_H
```