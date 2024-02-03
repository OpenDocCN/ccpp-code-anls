# `xmrig\src\base\tools\Span.h`

```cpp
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是基于希望它会有用而分发的，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。更多详情请参阅
 *   GNU通用公共许可证。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参阅<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_SPAN_H
#define XMRIG_SPAN_H


#include "3rdparty/epee/span.h"


namespace xmrig {


使用epee命名空间中的span类别
using Span = epee::span<const uint8_t>;


} /* namespace xmrig */


#endif /* XMRIG_SPAN_H */
```