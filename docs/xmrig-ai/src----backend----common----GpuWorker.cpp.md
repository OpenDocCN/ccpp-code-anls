# `xmrig\src\backend\common\GpuWorker.cpp`

```
/* XMRig
 * 版权所有（C）2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有（C）2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新分发或修改
 *   根据GNU通用公共许可证的条款发布，由
 *   自由软件基金会发布的许可证的第3版，或者
 *   （在您的选择）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */


#include "backend/common/GpuWorker.h"
#include "base/tools/Chrono.h"


// 构造函数，初始化 GPU 工作线程
xmrig::GpuWorker::GpuWorker(size_t id, int64_t affinity, int priority, uint32_t deviceIndex) : Worker(id, affinity, priority),
    m_deviceIndex(deviceIndex)
{
}


// 存储统计数据
void xmrig::GpuWorker::storeStats()
{
    // 获取当前未使用的索引
    const uint32_t index = m_index.load(std::memory_order_relaxed) ^ 1;

    // 填充该索引的数据
    m_hashCount[index] = m_count;
    m_timestamp[index] = Chrono::steadyMSecs();

    // 切换到该索引
    // 所有数据将在完成时存储在内存中，感谢std::memory_order_seq_cst
    m_index.fetch_xor(1, std::memory_order_seq_cst);
}


// 获取哈希率数据
void xmrig::GpuWorker::hashrateData(uint64_t &hashCount, uint64_t &timeStamp, uint64_t &rawHashes) const
{
    const uint32_t index = m_index.load(std::memory_order_relaxed);

    rawHashes = m_hashrateData.interpolate(timeStamp);
    hashCount = m_hashCount[index];
    timeStamp = m_timestamp[index];
}
```