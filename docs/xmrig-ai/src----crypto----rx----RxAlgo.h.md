# `xmrig\src\crypto\rx\RxAlgo.h`

```
/* XMRig
 * 版权所有（C）2018-2019 tevador     <tevador@gmail.com>
 * 版权所有（C）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（C）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以根据GNU通用公共许可证的条款重新分发或修改它，无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是基于希望它将是有用的，但没有任何保证；甚至没有暗示的保证适销性或特定用途的保证。有关更多详细信息，请参见GNU通用公共许可证。
 *
 * 您应该已经收到了GNU通用公共许可证的副本。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_RX_ALGO_H
#define XMRIG_RX_ALGO_H

#include <cstddef>
#include <cstdint>

#include "base/crypto/Algorithm.h"

// 定义 RandomX_ConfigurationBase 结构体
struct RandomX_ConfigurationBase;

// 命名空间 xmrig
namespace xmrig
{
    // RxAlgo 类
    class RxAlgo
    {
    public:
        // 应用算法
        static Algorithm::Id apply(Algorithm::Id algorithm);
        // 获取算法的基本配置
        static const RandomX_ConfigurationBase *base(Algorithm::Id algorithm);
        // 获取程序计数
        static uint32_t programCount(Algorithm::Id algorithm);
        // 获取程序迭代次数
        static uint32_t programIterations(Algorithm::Id algorithm);
        // 获取程序大小
        static uint32_t programSize(Algorithm::Id algorithm);
        // 获取版本
        static uint32_t version(Algorithm::Id algorithm);

        // 内联函数，返回算法的ID
        static inline Algorithm::Id id(Algorithm::Id algorithm)
        {
            // 如果算法是 RX_SFX，则返回 RX_0
            if (algorithm == Algorithm::RX_SFX) {
                return Algorithm::RX_0;
            }

            return algorithm;
        }
    };
} /* namespace xmrig */

#endif /* XMRIG_RX_ALGO_H */
```