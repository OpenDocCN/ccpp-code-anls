# `xmrig\src\backend\opencl\runners\tools\OclSharedState.cpp`

```
/*
 * XMRig
 * 版权所有 2010      Jeff Garzik <jgarzik@pobox.com>
 * 版权所有 2012-2014 pooler      <pooler@litecoinpool.org>
 * 版权所有 2014      Lucas Jones <https://github.com/lucasjones>
 * 版权所有 2014-2016 Wolf9466    <https://github.com/OhGodAPet>
 * 版权所有 2016      Jay D Dee   <jayddee246@gmail.com>
 * 版权所有 2017-2018 XMR-Stak    <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * 版权所有 2018-2019 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2019 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改
 * 根据 GNU 通用公共许可证的条款发布，
 * 由自由软件基金会发布的版本 3 或
 * （根据您的选择）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证。
 * 有关更多详细信息，请参见
 * GNU 通用公共许可证。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "backend/opencl/runners/tools/OclSharedState.h"
#include "backend/opencl/runners/tools/OclSharedData.h"

#include <cassert>
#include <map>

namespace xmrig {
    // 创建静态映射表
    static std::map<uint32_t, OclSharedData> map;
} // namespace xmrig

// 获取指定索引的共享数据
xmrig::OclSharedData &xmrig::OclSharedState::get(uint32_t index)
{
    return map[index];
}

// 释放所有共享数据
void xmrig::OclSharedState::release()
{
    for (auto &kv : map) {
        kv.second.release();
    }
    // 清空映射表
    map.clear();
}

// 启动 OpenCL 线程和作业
void xmrig::OclSharedState::start(const std::vector<OclLaunchData> &threads, const Job &job)
{
    // 确保映射表为空
    assert(map.empty());

    for (const auto &data : threads) {
        // 获取设备索引对应的共享数据
        auto &sharedData = map[data.device.index()];
        // 增加共享数据的计数
        ++sharedData;
    }
}
# 如果定义了 XMRIG_ALGO_RANDOMX 宏
        if (data.algorithm.family() == Algorithm::RANDOM_X) {
            # 创建数据集，根据数据上下文、作业和线程是否为数据集主机
            sharedData.createDataset(data.ctx, job, data.thread.isDatasetHost());
        }
# 结束条件编译指令
# 结束函数定义
```