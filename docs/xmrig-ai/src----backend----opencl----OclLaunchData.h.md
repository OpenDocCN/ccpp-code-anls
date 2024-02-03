# `xmrig\src\backend\opencl\OclLaunchData.h`

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

#ifndef XMRIG_OCLLAUNCHDATA_H
#define XMRIG_OCLLAUNCHDATA_H


#include "backend/opencl/OclThread.h"
#include "backend/opencl/runners/tools/OclSharedData.h"
#include "backend/opencl/wrappers/OclDevice.h"
#include "backend/opencl/wrappers/OclPlatform.h"
#include "base/crypto/Algorithm.h"
#include "crypto/common/Nonce.h"


using cl_context = struct _cl_context *;


namespace xmrig {


class OclConfig;
class Miner;


class OclLaunchData
{
public:
    // 构造函数，初始化 OclLaunchData 对象
    OclLaunchData(const Miner *miner, const Algorithm &algorithm, const OclConfig &config, const OclPlatform &platform, const OclThread &thread, const OclDevice &device, int64_t affinity);

    // 检查两个 OclLaunchData 对象是否相等
    bool isEqual(const OclLaunchData &other) const;
    // 返回后端类型为 OPENCL
    inline constexpr static Nonce::Backend backend() { return Nonce::OPENCL; }

    // 比较两个 OclLaunchData 对象是否不相等
    inline bool operator!=(const OclLaunchData &other) const    { return !isEqual(other); }
    // 比较两个 OclLaunchData 对象是否相等
    inline bool operator==(const OclLaunchData &other) const    { return isEqual(other); }

    // 返回标签
    static const char *tag();

    // OpenCL 上下文
    cl_context ctx = nullptr;
    // 算法
    const Algorithm algorithm;
    // 是否缓存
    const bool cache;
    // 亲和性
    const int64_t affinity;
    // 矿工
    const Miner *miner;
    // OpenCL 设备
    const OclDevice device;
    // OpenCL 平台
    const OclPlatform platform;
    // OpenCL 线程
    const OclThread thread;
    // 基准大小
    const uint32_t benchSize = 0;
}; 
// 结束了 xmrig 命名空间的定义

} // namespace xmrig
// 定义了 xmrig 命名空间

#endif /* XMRIG_OCLLAUNCHDATA_H */
// 结束了 XMRIG_OCLLAUNCHDATA_H 的条件编译
```