# `xmrig\src\backend\common\interfaces\IRxListener.h`

```
/* XMRig
 * 版权所有 (c) 2018-2019 tevador     <tevador@gmail.com>
 * 版权所有 (c) 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以根据 GNU 通用公共许可证的条款重新分发和/或修改
 *   它，无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，但没有任何担保；甚至没有暗示的担保。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参阅 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_IRXLISTENER_H
#define XMRIG_IRXLISTENER_H


#include "base/tools/Object.h"


namespace xmrig {


class IRxListener
{
public:
    XMRIG_DISABLE_COPY_MOVE(IRxListener)

    IRxListener()           = default;
    virtual ~IRxListener()  = default;

#   ifdef XMRIG_ALGO_RANDOMX
    // 当数据集准备就绪时调用的虚函数
    virtual void onDatasetReady() = 0;
#   endif
};


} /* namespace xmrig */


#endif // XMRIG_IRXLISTENER_H
```