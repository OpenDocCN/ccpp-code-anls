# `xmrig\src\backend\cuda\runners\CudaKawPowRunner.cpp`

```
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款
 *   它，无论是许可证的第3版还是
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是在希望它有用的情况下分发的，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "backend/cuda/runners/CudaKawPowRunner.h"
#include "3rdparty/libethash/data_sizes.h"
#include "backend/cuda/CudaLaunchData.h"
#include "backend/cuda/wrappers/CudaLib.h"
#include "base/io/log/Log.h"
#include "base/io/log/Tags.h"
#include "base/net/stratum/Job.h"
#include "base/tools/Chrono.h"
#include "crypto/kawpow/KPCache.h"
#include "crypto/kawpow/KPHash.h"

// 实现CudaKawPowRunner类的构造函数
xmrig::CudaKawPowRunner::CudaKawPowRunner(size_t index, const CudaLaunchData &data) :
    CudaBaseRunner(index, data)
{
}

// 实现CudaKawPowRunner类的run方法
bool xmrig::CudaKawPowRunner::run(uint32_t /*startNonce*/, uint32_t *rescount, uint32_t *resnonce)
{
    return callWrapper(CudaLib::kawPowHash(m_ctx, m_jobBlob, m_target, rescount, resnonce, &m_skippedHashes));
}

// 实现CudaKawPowRunner类的set方法
bool xmrig::CudaKawPowRunner::set(const Job &job, uint8_t *blob)
{
    if (!CudaBaseRunner::set(job, blob)) {
        return false;
    }

    m_jobBlob = blob;

    const uint64_t height = job.height();
    const uint32_t epoch = height / KPHash::EPOCH_LENGTH;

    KPCache& cache = KPCache::s_cache;
    {
        std::lock_guard<std::mutex> lock(KPCache::s_cacheMutex);
        cache.init(epoch);
    }
}
    // 获取当前时间的毫秒数
    const uint64_t start_ms = Chrono::steadyMSecs();

    // 使用 CUDA 库准备 KawPow 算法所需的数据，返回操作结果
    const bool result = CudaLib::kawPowPrepare(m_ctx, cache.data(), cache.size(), cache.l1_cache(), KPCache::dag_size(epoch), height, dag_sizes);
    // 如果操作结果为假，记录错误日志
    if (!result) {
        LOG_ERR("%s " YELLOW("KawPow") RED(" failed to initialize DAG: ") RED_BOLD("%s"), Tags::nvidia(), CudaLib::lastError(m_ctx));
    }
    // 如果操作结果为真，计算时间差并记录信息日志
    else {
        const int64_t dt = Chrono::steadyMSecs() - start_ms;
        // 如果时间差大于1000毫秒，记录信息日志
        if (dt > 1000) {
            LOG_INFO("%s " YELLOW("KawPow") " DAG for epoch " WHITE_BOLD("%u") " calculated " BLACK_BOLD("(%" PRIu64 "ms)"), Tags::nvidia(), epoch, dt);
        }
    }

    // 返回操作结果
    return result;
# 定义了一个名为xmrig的命名空间，其中包含了CudaKawPowRunner类
void xmrig::CudaKawPowRunner::jobEarlyNotification(const Job&)
{
    # 调用CudaLib命名空间中的kawPowStopHash函数，传入m_ctx参数
    CudaLib::kawPowStopHash(m_ctx);
}
```