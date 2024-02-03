# `xmrig\src\backend\common\GpuWorker.h`

```cpp
/*
 * XMRig
 * 版权所有（c）2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款
 * 许可证的版本，或（在您的选择）任何以后的版本。
 *
 * 本程序是希望它有用的
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_GPUWORKER_H
#define XMRIG_GPUWORKER_H


#include <atomic>


#include "backend/common/HashrateInterpolator.h"
#include "backend/common/Worker.h"


namespace xmrig {


class GpuWorker : public Worker
{
public:
    // 构造函数，初始化 GPU 工作器
    GpuWorker(size_t id, int64_t affinity, int priority, uint32_t m_deviceIndex);

protected:
    // 覆盖基类的 memory() 方法，返回空指针
    inline const VirtualMemory *memory() const override     { return nullptr; }
    // 返回设备索引
    inline uint32_t deviceIndex() const                     { return m_deviceIndex; }

    // 获取哈希率数据
    void hashrateData(uint64_t &hashCount, uint64_t &timeStamp, uint64_t &rawHashes) const override;

protected:
    // 存储统计数据
    void storeStats();

    const uint32_t m_deviceIndex; // 设备索引
    HashrateInterpolator m_hashrateData; // 哈希率插值器
    std::atomic<uint32_t> m_index   = {}; // 原子整型变量
    uint64_t m_hashCount[2]         = {}; // 哈希计数数组
    uint64_t m_timestamp[2]         = {}; // 时间戳数组
};


} // namespace xmrig


#endif /* XMRIG_GPUWORKER_H */
```