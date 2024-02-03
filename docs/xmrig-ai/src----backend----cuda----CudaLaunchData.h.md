# `xmrig\src\backend\cuda\CudaLaunchData.h`

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
 * 本程序是希望它有用
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。有关更多详细信息
 * 请参阅GNU通用公共许可证。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_CUDALAUNCHDATA_H
#define XMRIG_CUDALAUNCHDATA_H


#include "backend/cuda/CudaThread.h"
#include "base/crypto/Algorithm.h"
#include "crypto/common/Nonce.h"


namespace xmrig {


class CudaDevice;
class Miner;


class CudaLaunchData
{
public:
    // 构造函数，初始化CudaLaunchData对象
    CudaLaunchData(const Miner *miner, const Algorithm &algorithm, const CudaThread &thread, const CudaDevice &device);

    // 检查两个CudaLaunchData对象是否相等
    bool isEqual(const CudaLaunchData &other) const;

    // 返回后端类型为CUDA
    inline constexpr static Nonce::Backend backend() { return Nonce::CUDA; }

    // 重载不等于运算符
    inline bool operator!=(const CudaLaunchData &other) const    { return !isEqual(other); }
    // 重载等于运算符
    inline bool operator==(const CudaLaunchData &other) const    { return isEqual(other); }

    // 返回标签
    static const char *tag();

    // 算法
    const Algorithm algorithm;
    // CUDA设备
    const CudaDevice &device;
    // CUDA线程
    const CudaThread thread;
    // 亲和性
    const int64_t affinity;
    // 矿工
    const Miner *miner;
    // 基准大小
    const uint32_t benchSize = 0;
};


} // namespace xmrig


#endif /* XMRIG_OCLLAUNCHDATA_H */
```