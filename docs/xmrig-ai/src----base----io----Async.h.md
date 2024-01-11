# `xmrig\src\base\io\Async.h`

```
/* XMRig
 * 版权所有（c）2015-2020 libuv 项目贡献者。
 * 版权所有（c）2020      cohcho      <https://github.com/cohcho>
 * 版权所有（c）2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   许可证的版本为3，或者（在您的选择下）任何以后的版本。
 *
 *   本程序是基于希望它将是有用的分发的，
 *   但没有任何保证；甚至没有暗示的保证适销性或特定用途的保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_ASYNC_H
#define XMRIG_ASYNC_H


#include "base/tools/Object.h"


#include <functional>


namespace xmrig {


class AsyncPrivate;
class IAsyncListener;


class Async
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(Async)

    using Callback = std::function<void()>;

    Async(Callback callback); // 构造函数，接受一个回调函数作为参数
    Async(IAsyncListener *listener); // 构造函数，接受一个IAsyncListener指针作为参数
    ~Async(); // 析构函数

    void send(); // 发送函数

private:
    AsyncPrivate *d_ptr; // 私有成员指针
};


} // namespace xmrig


#endif /* XMRIG_ASYNC_H */
```