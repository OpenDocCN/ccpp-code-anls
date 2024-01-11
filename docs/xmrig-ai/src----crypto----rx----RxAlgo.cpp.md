# `xmrig\src\crypto\rx\RxAlgo.cpp`

```
/* XMRig
 * 版权所有 (c) 2018-2019 tevador     <tevador@gmail.com>
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以根据 GNU 通用公共许可证的条款重新分发或修改它，
 *   其中规定了由自由软件基金会发布的版本 3 的许可证，或者
 *   （按您的选择）任何以后的版本。
 *
 *   本程序是希望它有用，但没有任何担保；甚至没有暗示的担保。
 *   有关更多详细信息，请参见 GNU 通用公共许可证。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#include "crypto/randomx/randomx.h"
#include "crypto/rx/RxAlgo.h"


xmrig::Algorithm::Id xmrig::RxAlgo::apply(Algorithm::Id algorithm)
{
    // 应用随机X配置
    randomx_apply_config(*base(algorithm));

    // 返回算法
    return algorithm;
}


const RandomX_ConfigurationBase *xmrig::RxAlgo::base(Algorithm::Id algorithm)
{
    switch (algorithm) {
    case Algorithm::RX_WOW:
        return &RandomX_WowneroConfig;

    case Algorithm::RX_ARQ:
        return &RandomX_ArqmaConfig;

    case Algorithm::RX_GRAFT:
        return &RandomX_GraftConfig;

    case Algorithm::RX_SFX:
        return &RandomX_SafexConfig;

    case Algorithm::RX_KEVA:
        return &RandomX_KevaConfig;

    default:
        break;
    }

    return &RandomX_MoneroConfig;
}


uint32_t xmrig::RxAlgo::version(Algorithm::Id algorithm)
{
    // 如果算法是RX_WOW，则返回103，否则返回104
    return algorithm == Algorithm::RX_WOW ? 103 : 104;
}


uint32_t xmrig::RxAlgo::programCount(Algorithm::Id algorithm)
{
    // 返回算法的程序计数
    return base(algorithm)->ProgramCount;
}


uint32_t xmrig::RxAlgo::programIterations(Algorithm::Id algorithm)
{
    // 返回算法的程序迭代次数
    return base(algorithm)->ProgramIterations;
}
# 返回指定算法的程序大小
uint32_t xmrig::RxAlgo::programSize(Algorithm::Id algorithm)
{
    # 调用base函数获取指定算法的基本信息，并返回其程序大小
    return base(algorithm)->ProgramSize;
}
```