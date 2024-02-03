# `xmrig\src\base\kernel\interfaces\IBaseListener.h`

```cpp
/* XMRig
 * 版权所有 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以根据
 *   GNU 通用公共许可证的条款重新分发或修改
 *   它，由
 *   自由软件基金会发布的版本 3 或
 *   （根据您的选择）任何更高版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有
 *   对特定目的的隐含保证。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参阅 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_IBASELISTENER_H
#define XMRIG_IBASELISTENER_H


#include "base/tools/Object.h"


namespace xmrig {


class Config;


class IBaseListener
{
public:
    XMRIG_DISABLE_COPY_MOVE(IBaseListener)

    IBaseListener()             = default;
    virtual ~IBaseListener()    = default;

    virtual void onConfigChanged(Config *config, Config *previousConfig) = 0;
};


} /* namespace xmrig */


#endif // XMRIG_IBASELISTENER_H
```