# `xmrig\src\backend\common\interfaces\IBenchListener.h`

```
/*
 * XMRig
 * 版权所有（c）2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款
 * 许可证的版本，或（在您的选择）任何以后的版本。
 *
 * 本程序是希望它有用的
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。有关更多详细信息
 * 请参阅GNU通用公共许可证。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_IBENCHLISTENER_H
#define XMRIG_IBENCHLISTENER_H


#include "base/tools/Object.h"


namespace xmrig {


class IBackend;


class IBenchListener
{
public:
    XMRIG_DISABLE_COPY_MOVE(IBenchListener)

    IBenchListener()            = default;
    virtual ~IBenchListener()   = default;

    virtual void onBenchDone(uint64_t result, uint64_t diff, uint64_t ts)               = 0;
    virtual void onBenchReady(uint64_t ts, uint32_t threads, const IBackend *backend)   = 0;
};


} /* namespace xmrig */


#endif // XMRIG_IBENCHLISTENER_H
```