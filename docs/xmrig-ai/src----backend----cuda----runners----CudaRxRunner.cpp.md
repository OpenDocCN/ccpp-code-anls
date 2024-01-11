# `xmrig\src\backend\cuda\runners\CudaRxRunner.cpp`

```
/* XMRig
 * 版权声明
 * 该程序是自由软件：您可以重新发布或修改它，遵循 GNU 通用公共许可证的条款，可以选择遵循许可证的第3版或者之后的版本。
 * 该程序是基于希望它能有用而发布的，但没有任何担保；甚至没有暗示的担保。更多详情请参阅 GNU 通用公共许可证。
 * 您应该已经收到了 GNU 通用公共许可证的副本。如果没有，请访问 <http://www.gnu.org/licenses/>。
 */

#include "backend/cuda/runners/CudaRxRunner.h"
#include "backend/cuda/CudaLaunchData.h"
#include "backend/cuda/wrappers/CudaLib.h"
#include "base/net/stratum/Job.h"
#include "crypto/rx/Rx.h"
#include "crypto/rx/RxDataset.h"

// 构造函数，初始化 CudaRxRunner 对象
xmrig::CudaRxRunner::CudaRxRunner(size_t index, const CudaLaunchData &data) :
    CudaBaseRunner(index, data),  // 调用基类的构造函数
    m_datasetHost(data.thread.datasetHost() > 0)  // 初始化 m_datasetHost 成员变量
{
    // 计算强度
    m_intensity                   = m_data.thread.threads() * m_data.thread.blocks();
    // 计算 scratchpads 的大小
    const size_t scratchpads_size = m_intensity * m_data.algorithm.l3();
    // 计算 scratchpads 的数量
    const size_t num_scratchpads  = scratchpads_size / m_data.algorithm.l3();

    // 如果强度大于 scratchpads 的数量，则将强度设置为 scratchpads 的数量
    if (m_intensity > num_scratchpads) {
        m_intensity = num_scratchpads;
    }

    // 将强度调整为 32 的倍数
    m_intensity -= m_intensity % 32;
}
# 运行 CudaRxRunner 类的 run 方法，传入起始 nonce、结果计数和结果 nonce 的指针，返回运行结果
bool xmrig::CudaRxRunner::run(uint32_t startNonce, uint32_t *rescount, uint32_t *resnonce)
{
    # 调用 callWrapper 方法，传入 CudaLib::rxHash 方法的返回值作为参数
    return callWrapper(CudaLib::rxHash(m_ctx, startNonce, m_target, rescount, resnonce));
}

# 设置 CudaRxRunner 类的参数，传入作业和二进制数据，返回设置结果
bool xmrig::CudaRxRunner::set(const Job &job, uint8_t *blob)
{
    # 调用 CudaBaseRunner 类的 set 方法，传入作业和二进制数据，将返回结果保存到 rc 变量
    const bool rc = CudaBaseRunner::set(job, blob);
    # 如果 rc 为假或者 m_ready 为真，则返回 rc
    if (!rc || m_ready) {
        return rc;
    }
    # 调用 Rx::dataset 方法，传入作业和 0 作为参数，将返回结果保存到 dataset 变量
    auto dataset = Rx::dataset(job, 0);
    # 调用 callWrapper 方法，传入 CudaLib::rxPrepare 方法的返回值作为参数
    m_ready = callWrapper(CudaLib::rxPrepare(m_ctx, dataset->raw(), dataset->size(false), m_datasetHost, m_intensity));
    # 返回 m_ready
    return m_ready;
}
```