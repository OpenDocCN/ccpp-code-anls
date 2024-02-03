# `xmrig\src\crypto\cn\CnCtx.h`

```cpp
/* XMRig
 * 版权所有（c）2018      Lee Clagett <https://github.com/vtnerd>
 * 版权所有（c）2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以根据GNU通用公共许可证的条款重新分发或修改它，无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是基于希望它将是有用的分发的，但没有任何保证；甚至没有对特定目的的隐含保证。有关更多详细信息，请参见GNU通用公共许可证。
 *
 * 您应该已经收到了GNU通用公共许可证的副本。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_CN_CTX_H
#define XMRIG_CN_CTX_H

#include <cstddef>
#include <cstdint>

// 定义cryptonight_ctx结构体
struct cryptonight_ctx;

namespace xmrig
{
    // 定义CnCtx类
    class CnCtx
    {
    public:
        // 创建cryptonight_ctx对象
        static void create(cryptonight_ctx **ctx, uint8_t *memory, size_t size, size_t count);
        // 释放cryptonight_ctx对象
        static void release(cryptonight_ctx **ctx, size_t count);
    };
} /* namespace xmrig */

#endif /* XMRIG_CN_CTX_H */
```