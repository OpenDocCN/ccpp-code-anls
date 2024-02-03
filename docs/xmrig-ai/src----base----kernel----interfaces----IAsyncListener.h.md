# `xmrig\src\base\kernel\interfaces\IAsyncListener.h`

```cpp
/* XMRig
 * 版权所有 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以根据
 *   GNU 通用公共许可证的条款重新分发或修改
 *   由自由软件基金会发布的版本3的许可证，或者
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_IASYNCLISTENER_H
#define XMRIG_IASYNCLISTENER_H


#include "base/tools/Object.h"


namespace xmrig {


class Async;


class IAsyncListener
{
public:
    XMRIG_DISABLE_COPY_MOVE(IAsyncListener)

    // 默认构造函数
    IAsyncListener()            = default;
    // 虚析构函数
    virtual ~IAsyncListener()   = default;

    // 纯虚函数，用于异步操作的回调
    virtual void onAsync()      = 0;
};


} /* namespace xmrig */


#endif // XMRIG_IASYNCLISTENER_H
```