# `xmrig\src\backend\common\HashrateInterpolator.h`

```cpp
/* XMRig
 * 版权所有 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 *   无论是许可证的第 3 版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有适销性或特定用途的暗示保证。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_HASHRATE_INTERPOLATOR_H
#define XMRIG_HASHRATE_INTERPOLATOR_H


#include <mutex>
#include <deque>
#include <utility>


namespace xmrig {


class HashrateInterpolator
{
public:
    enum {
        LagMS = 4000,
    };

    // 插值函数，根据时间戳进行插值
    uint64_t interpolate(uint64_t timeStamp) const;
    // 添加数据点，包括计数和时间戳
    void addDataPoint(uint64_t count, uint64_t timeStamp);

private:
    // 用于线性插值过去数据的哈希率计数器缓冲区
    mutable std::mutex m_lock;
    std::deque<std::pair<uint64_t, uint64_t> > m_data;
};


} // namespace xmrig


#endif /* XMRIG_HASHRATE_INTERPOLATOR_H */
```