# `xmrig\src\crypto\cn\CnAlgo.h`

```cpp
/*
 * XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据GNU通用公共许可证的条款发布，由
 * 自由软件基金会发布的许可证的第3版，或者
 * （根据您的选择）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_CN_ALGO_H
#define XMRIG_CN_ALGO_H


#include <cstddef>
#include <cstdint>


#include "base/crypto/Algorithm.h"


namespace xmrig
{


template<Algorithm::Id ALGO = Algorithm::INVALID>
class CnAlgo
{
public:
    constexpr CnAlgo() {};

    // 返回算法的基础ID
    constexpr inline Algorithm::Id base() const  { static_assert(Algorithm::isCN(ALGO), "invalid CRYPTONIGHT algorithm"); return Algorithm::base(ALGO); }
    // 检查算法是否为CN_HEAVY类型
    constexpr inline bool isHeavy() const        { return Algorithm::family(ALGO) == Algorithm::CN_HEAVY; }
    // 检查算法是否为CN_R类型
    constexpr inline bool isR() const            { return ALGO == Algorithm::CN_R; }
    // 返回算法的内存大小
    constexpr inline size_t memory() const       { static_assert(Algorithm::isCN(ALGO), "invalid CRYPTONIGHT algorithm"); return Algorithm::l3(ALGO); }
    // 返回算法的迭代次数
    constexpr inline uint32_t iterations() const { static_assert(Algorithm::isCN(ALGO), "invalid CRYPTONIGHT algorithm"); return CN_ITER; }
    // 返回算法的掩码
    constexpr inline uint32_t mask() const       { return static_cast<uint32_t>(((memory() - 1) / 16) * 16); }
    // 返回算法的一半内存大小
    constexpr inline uint32_t half_mem() const   { return mask() < memory() / 2; }

    // 返回指定算法的迭代次数
    inline static uint32_t iterations(Algorithm::Id algo)
        # 根据不同的算法类型进行不同的处理
        switch (algo) {
        # 对于算法类型为CN_0、CN_1、CN_2、CN_R、CN_RTO，返回CN_ITER
        case Algorithm::CN_0:
        case Algorithm::CN_1:
        case Algorithm::CN_2:
        case Algorithm::CN_R:
        case Algorithm::CN_RTO:
            return CN_ITER;

        # 对于算法类型为CN_FAST、CN_HALF，待续...
        case Algorithm::CN_FAST:
        case Algorithm::CN_HALF:
// 如果定义了 XMRIG_ALGO_CN_LITE，则执行以下代码
        case Algorithm::CN_LITE_0:
        case Algorithm::CN_LITE_1:
// 如果定义了 XMRIG_ALGO_CN_HEAVY，则执行以下代码
        case Algorithm::CN_HEAVY_0:
        case Algorithm::CN_HEAVY_TUBE:
        case Algorithm::CN_HEAVY_XHV:
// 对于 CN_CCX 算法，返回 CN_ITER 的一半
        case Algorithm::CN_CCX:
            return CN_ITER / 2;
// 对于 CN_RWZ 和 CN_ZLS 算法，返回 0x60000
        case Algorithm::CN_RWZ:
        case Algorithm::CN_ZLS:
            return 0x60000;
// 对于 CN_XAO 和 CN_DOUBLE 算法，返回 CN_ITER 的两倍
        case Algorithm::CN_XAO:
        case Algorithm::CN_DOUBLE:
            return CN_ITER * 2;
// 如果定义了 XMRIG_ALGO_CN_PICO，则执行以下代码
        case Algorithm::CN_PICO_0:
        case Algorithm::CN_PICO_TLO:
            return CN_ITER / 8;
// 如果定义了 XMRIG_ALGO_CN_FEMTO，则执行以下代码
        case Algorithm::CN_UPX2:
            return CN_ITER / 32;
// 默认情况下，返回 0
        default:
            break;
        }
// 返回 0
        return 0;
    }

// 返回算法的掩码值
    inline static uint32_t mask(Algorithm::Id algo)
    {
// 如果定义了 XMRIG_ALGO_CN_PICO，并且算法为 CN_PICO_0，则执行以下代码
        if (algo == Algorithm::CN_PICO_0) {
            return 0x1FFF0;
        }
// 如果定义了 XMRIG_ALGO_CN_FEMTO，并且算法为 CN_UPX2，则执行以下代码
        if (algo == Algorithm::CN_UPX2) {
            return 0x1FFF0;
        }
// 如果定义了 XMRIG_ALGO_GHOSTRIDER，并且算法为 CN_GR_1，则执行以下代码
        if (algo == Algorithm::CN_GR_1) {
            return 0x3FFF0;
        }
// 如果定义了 XMRIG_ALGO_GHOSTRIDER，并且算法为 CN_GR_5，则执行以下代码
        if (algo == Algorithm::CN_GR_5) {
            return 0x1FFF0;
        }
// 返回算法的掩码值
        return ((Algorithm::l3(algo) - 1) / 16) * 16;
    }

private:
    constexpr const static uint32_t CN_ITER = 0x80000;
};


template<> constexpr inline uint32_t CnAlgo<Algorithm::CN_FAST>::iterations() const         { return CN_ITER / 2; }
template<> constexpr inline uint32_t CnAlgo<Algorithm::CN_HALF>::iterations() const         { return CN_ITER / 2; }
template<> constexpr inline uint32_t CnAlgo<Algorithm::CN_LITE_0>::iterations() const       { return CN_ITER / 2; }
template<> constexpr inline uint32_t CnAlgo<Algorithm::CN_LITE_1>::iterations() const       { return CN_ITER / 2; }
// 返回 CN_HEAVY_0 算法的迭代次数
template<> constexpr inline uint32_t CnAlgo<Algorithm::CN_HEAVY_0>::iterations() const      { return CN_ITER / 2; }
// 返回 CN_HEAVY_TUBE 算法的迭代次数
template<> constexpr inline uint32_t CnAlgo<Algorithm::CN_HEAVY_TUBE>::iterations() const   { return CN_ITER / 2; }
// 返回 CN_HEAVY_XHV 算法的迭代次数
template<> constexpr inline uint32_t CnAlgo<Algorithm::CN_HEAVY_XHV>::iterations() const    { return CN_ITER / 2; }
// 返回 CN_XAO 算法的迭代次数
template<> constexpr inline uint32_t CnAlgo<Algorithm::CN_XAO>::iterations() const          { return CN_ITER * 2; }
// 返回 CN_DOUBLE 算法的迭代次数
template<> constexpr inline uint32_t CnAlgo<Algorithm::CN_DOUBLE>::iterations() const       { return CN_ITER * 2; }
// 返回 CN_RWZ 算法的迭代次数
template<> constexpr inline uint32_t CnAlgo<Algorithm::CN_RWZ>::iterations() const          { return 0x60000; }
// 返回 CN_ZLS 算法的迭代次数
template<> constexpr inline uint32_t CnAlgo<Algorithm::CN_ZLS>::iterations() const          { return 0x60000; }
// 返回 CN_PICO_0 算法的迭代次数
template<> constexpr inline uint32_t CnAlgo<Algorithm::CN_PICO_0>::iterations() const       { return CN_ITER / 8; }
// 返回 CN_PICO_TLO 算法的迭代次数
template<> constexpr inline uint32_t CnAlgo<Algorithm::CN_PICO_TLO>::iterations() const     { return CN_ITER / 8; }
// 返回 CN_CCX 算法的迭代次数
template<> constexpr inline uint32_t CnAlgo<Algorithm::CN_CCX>::iterations() const          { return CN_ITER / 2; }
// 返回 CN_UPX2 算法的迭代次数
template<> constexpr inline uint32_t CnAlgo<Algorithm::CN_UPX2>::iterations() const         { return CN_ITER / 32; }

// 返回 CN_PICO_0 算法的掩码
template<> constexpr inline uint32_t CnAlgo<Algorithm::CN_PICO_0>::mask() const             { return 0x1FFF0; }
// 返回 CN_UPX2 算法的掩码
template<> constexpr inline uint32_t CnAlgo<Algorithm::CN_UPX2>::mask() const               { return 0x1FFF0; }

#ifdef XMRIG_ALGO_GHOSTRIDER
// 返回 CN_GR_0 算法的迭代次数
template<> constexpr inline uint32_t CnAlgo<Algorithm::CN_GR_0>::iterations() const         { return CN_ITER / 4; }
// 返回 CN_GR_1 算法的迭代次数
template<> constexpr inline uint32_t CnAlgo<Algorithm::CN_GR_1>::iterations() const         { return CN_ITER / 4; }
// 返回 CN_GR_2 算法的迭代次数
template<> constexpr inline uint32_t CnAlgo<Algorithm::CN_GR_2>::iterations() const         { return CN_ITER / 2; }
// 返回 CN_GR_3 算法的迭代次数
template<> constexpr inline uint32_t CnAlgo<Algorithm::CN_GR_3>::iterations() const         { return CN_ITER / 2; }
// 返回CN_GR_4算法的迭代次数
template<> constexpr inline uint32_t CnAlgo<Algorithm::CN_GR_4>::iterations() const         { return CN_ITER / 8; }
// 返回CN_GR_5算法的迭代次数
template<> constexpr inline uint32_t CnAlgo<Algorithm::CN_GR_5>::iterations() const         { return CN_ITER / 8; }

// 返回CN_GR_1算法的掩码
template<> constexpr inline uint32_t CnAlgo<Algorithm::CN_GR_1>::mask() const               { return 0x3FFF0; }
// 返回CN_GR_5算法的掩码
template<> constexpr inline uint32_t CnAlgo<Algorithm::CN_GR_5>::mask() const               { return 0x1FFF0; }
#endif


} /* namespace xmrig */


#endif /* XMRIG_CN_ALGO_H */
```