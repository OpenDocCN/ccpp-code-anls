# `xmrig\src\backend\opencl\OclLaunchData.cpp`

```cpp
/*
 * XMRig
 * 版权所有 2010      Jeff Garzik <jgarzik@pobox.com>
 * 版权所有 2012-2014 pooler      <pooler@litecoinpool.org>
 * 版权所有 2014      Lucas Jones <https://github.com/lucasjones>
 * 版权所有 2014-2016 Wolf9466    <https://github.com/OhGodAPet>
 * 版权所有 2016      Jay D Dee   <jayddee246@gmail.com>
 * 版权所有 2017-2018 XMR-Stak    <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * 版权所有 2018      Lee Clagett <https://github.com/vtnerd>
 * 版权所有 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改
 * 根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 * 要么是许可证的第3版，要么（在您的选择下）是任何更高版本。
 *
 * 本程序是希望它有用，但没有任何担保；甚至没有暗示的担保。
 * 有关更多详细信息，请参见 GNU 通用公共许可证。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */


#include "backend/opencl/OclLaunchData.h"
#include "backend/common/Tags.h"
#include "backend/opencl/OclConfig.h"


xmrig::OclLaunchData::OclLaunchData(const Miner *miner, const Algorithm &algorithm, const OclConfig &config, const OclPlatform &platform, const OclThread &thread, const OclDevice &device, int64_t affinity) :
    algorithm(algorithm),  // 初始化算法
    cache(config.isCacheEnabled()),  // 初始化缓存
    affinity(affinity),  // 初始化亲和性
    miner(miner),  // 初始化矿工
    device(device),  // 初始化设备
    platform(platform),  // 初始化平台
    thread(thread)  // 初始化线程
{
}


bool xmrig::OclLaunchData::isEqual(const OclLaunchData &other) const
{
    return (other.algorithm == algorithm &&  // 比较算法是否相等
            other.thread    == thread);  // 比较线程是否相等
}


const char *xmrig::OclLaunchData::tag()
{
    # 调用 ocl_tag 函数并返回其结果
    return ocl_tag();
# 闭合前面的函数定义
```