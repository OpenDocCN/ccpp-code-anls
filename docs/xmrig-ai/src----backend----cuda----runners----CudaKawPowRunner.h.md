# `xmrig\src\backend\cuda\runners\CudaKawPowRunner.h`

```cpp
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   许可证的版本为3，或者（根据您的选择）任何以后的版本。
 *
 *   本程序是希望它有用的分发，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。更多详情请参见
 *   GNU通用公共许可证。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_CUDAKAWPOWRUNNER_H
#define XMRIG_CUDAKAWPOWRUNNER_H


#include "backend/cuda/runners/CudaBaseRunner.h"


namespace xmrig {


class CudaKawPowRunner : public CudaBaseRunner
{
public:
    // 构造函数，初始化CudaKawPowRunner对象
    CudaKawPowRunner(size_t index, const CudaLaunchData &data);

protected:
    // 运行KawPow算法，返回是否成功以及结果计数和结果nonce
    bool run(uint32_t startNonce, uint32_t *rescount, uint32_t *resnonce) override;
    // 设置作业和blob数据，返回是否成功
    bool set(const Job &job, uint8_t *blob) override;
    // 返回已处理的哈希数量
    size_t processedHashes() const override { return intensity() - m_skippedHashes; }
    // 作业提前通知
    void jobEarlyNotification(const Job&) override;

private:
    // 作业blob数据
    uint8_t* m_jobBlob = nullptr;
    // 跳过的哈希数量
    uint32_t m_skippedHashes = 0;
};


} /* namespace xmrig */


#endif // XMRIG_CUDAKAWPOWRUNNER_H
```