# `xmrig\src\backend\cpu\CpuLaunchData.h`

```
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

#ifndef XMRIG_CPULAUNCHDATA_H
#define XMRIG_CPULAUNCHDATA_H


#include "base/crypto/Algorithm.h"
#include "crypto/cn/CnHash.h"
#include "crypto/common/Assembly.h"
#include "crypto/common/Nonce.h"


namespace xmrig {


class CpuConfig;
class CpuThread;
class Miner;


class CpuLaunchData
{
public:
    // 构造函数，初始化 CPU 启动数据
    CpuLaunchData(const Miner *miner, const Algorithm &algorithm, const CpuConfig &config, const CpuThread &thread, size_t threads, const std::vector<int64_t>& affinities);

    // 比较两个 CPU 启动数据是否相等
    bool isEqual(const CpuLaunchData &other) const;
    // 获取算法变体
    CnHash::AlgoVariant av() const;

    // 获取后端类型为 CPU 的 Nonce
    inline constexpr static Nonce::Backend backend()            { return Nonce::CPU; }
    // 重载不等于运算符，调用 isEqual 方法取反
    inline bool operator!=(const CpuLaunchData &other) const    { return !isEqual(other); }
    // 重载等于运算符，调用 isEqual 方法
    inline bool operator==(const CpuLaunchData &other) const    { return isEqual(other); }

    // 返回静态成员变量 tag
    static const char *tag();

    // 声明常量成员变量 algorithm, assembly, hugePages, hwAES, yield, priority, affinity, miner, threads, intensity, affinities
    const Algorithm algorithm;
    const Assembly assembly;
    const bool hugePages;
    const bool hwAES;
    const bool yield;
    const int priority;
    const int64_t affinity;
    const Miner *miner;
    const size_t threads;
    const uint32_t intensity;
    const std::vector<int64_t> affinities;
}; // 结束命名空间 xmrig

} // 结束命名空间 xmrig

#endif /* XMRIG_CPULAUNCHDATA_H */ // 结束条件编译指令
```