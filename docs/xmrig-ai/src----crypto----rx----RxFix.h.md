# `xmrig\src\crypto\rx\RxFix.h`

```cpp
/*
 * XMRig
 * 版权所有 (c) 2018-2019 tevador     <tevador@gmail.com>
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以根据 GNU 通用公共许可证的条款重新分发或修改它，
 * 该许可证由自由软件基金会发布，无论是许可证的第3版，还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用，但没有任何担保；甚至没有暗示的担保适用于特定目的。有关更多详细信息，请参见 GNU 通用公共许可证。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本，如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_RXFIX_H
#define XMRIG_RXFIX_H

#include <utility>

namespace xmrig
{
    // 命名空间 xmrig

    class RxFix
    {
    public:
        // 设置主循环边界
        static void setMainLoopBounds(const std::pair<const void *, const void *> &bounds);
        // 设置主循环异常帧
        static void setupMainLoopExceptionFrame();
    };

} /* namespace xmrig */

#endif /* XMRIG_RXFIX_H */
```