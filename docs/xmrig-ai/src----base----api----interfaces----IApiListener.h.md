# `xmrig\src\base\api\interfaces\IApiListener.h`

```cpp
/* XMRig
 * 版权所有 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据由自由软件基金会发布的 GNU 通用公共许可证的条款
 *   它，无论是许可证的第 3 版还是
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

#ifndef XMRIG_IAPILISTENER_H
#define XMRIG_IAPILISTENER_H


#include "base/tools/Object.h"


namespace xmrig {


class IApiRequest;


class IApiListener
{
public:
    XMRIG_DISABLE_COPY_MOVE(IApiListener)

    IApiListener()          = default;
    virtual ~IApiListener() = default;

#   ifdef XMRIG_FEATURE_API
    virtual void onRequest(IApiRequest &request) = 0;
#   endif
};


} /* namespace xmrig */


#endif // XMRIG_IAPILISTENER_H
```