# `xmrig\src\backend\cuda\CudaLaunchData.cpp`

```cpp
/*
 * XMRig
 * 版权所有（c）2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款，
 * 无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用的分发，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。更多细节请参见
 * GNU通用公共许可证。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "backend/cuda/CudaLaunchData.h"
#include "backend/common/Tags.h"

// 构造函数，初始化 CUDA 启动数据
xmrig::CudaLaunchData::CudaLaunchData(const Miner *miner, const Algorithm &algorithm, const CudaThread &thread, const CudaDevice &device) :
    algorithm(algorithm),  // 初始化算法
    device(device),  // 初始化设备
    thread(thread),  // 初始化线程
    affinity(thread.affinity()),  // 初始化亲和性
    miner(miner)  // 初始化矿工
{
}

// 比较函数，判断两个 CUDA 启动数据是否相等
bool xmrig::CudaLaunchData::isEqual(const CudaLaunchData &other) const
{
    return (other.algorithm.family() == algorithm.family() &&  // 判断算法家族是否相等
            other.algorithm.l3()     == algorithm.l3() &&  // 判断算法 L3 是否相等
            other.thread             == thread);  // 判断线程是否相等
}

// 返回 CUDA 标签
const char *xmrig::CudaLaunchData::tag()
{
    return cuda_tag();  // 返回 CUDA 标签
}
```