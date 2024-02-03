# `xmrig\src\backend\common\HashrateInterpolator.cpp`

```cpp
/* XMRig
 * 版权所有 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证适用于特定目的。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */


#include "backend/common/HashrateInterpolator.h"


uint64_t xmrig::HashrateInterpolator::interpolate(uint64_t timeStamp) const
{
    // 减去延迟时间
    timeStamp -= LagMS;

    // 加锁，保护共享数据
    std::lock_guard<std::mutex> l(m_lock);

    // 获取数据点数量
    const size_t N = m_data.size();

    // 如果数据点数量小于2，返回0
    if (N < 2) {
        return 0;
    }

    // 遍历数据点
    for (size_t i = 0; i < N - 1; ++i) {
        const auto& a = m_data[i];
        const auto& b = m_data[i + 1];

        // 如果时间戳在两个数据点之间，进行插值计算
        if (a.second <= timeStamp && timeStamp <= b.second) {
            return a.first + static_cast<int64_t>(b.first - a.first) * (timeStamp - a.second) / (b.second - a.second);
        }
    }

    return 0;
}

void xmrig::HashrateInterpolator::addDataPoint(uint64_t count, uint64_t timeStamp)
{
    // 加锁，保护共享数据
    std::lock_guard<std::mutex> l(m_lock);

    // 清理旧数据
    while (!m_data.empty() && (timeStamp - m_data.front().second > LagMS * 2)) {
        m_data.pop_front();
    }

    // 添加新的数据点
    m_data.emplace_back(count, timeStamp);
}
```