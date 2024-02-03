# `xmrig\src\backend\cuda\runners\CudaBaseRunner.cpp`

```cpp
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
 * 根据 GNU 通用公共许可证的条款发布，由
 * 自由软件基金会发布的版本 3 或
 * （根据您的选择）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU 通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */


#include "backend/cuda/runners/CudaBaseRunner.h"
#include "backend/cuda/wrappers/CudaLib.h"
#include "backend/cuda/CudaLaunchData.h"
#include "backend/common/Tags.h"
#include "base/io/log/Log.h"
#include "base/net/stratum/Job.h"


// 初始化 CUDA 基础运行程序
xmrig::CudaBaseRunner::CudaBaseRunner(size_t id, const CudaLaunchData &data) :
    m_data(data),
    m_threadId(id)
{
}


// 释放 CUDA 基础运行程序
xmrig::CudaBaseRunner::~CudaBaseRunner()
{
    CudaLib::release(m_ctx);
}


// 初始化 CUDA 基础运行程序
bool xmrig::CudaBaseRunner::init()
{
    // 分配 CUDA 上下文
    m_ctx = CudaLib::alloc(m_data.thread.index(), m_data.thread.bfactor(), m_data.thread.bsleep());
    // 调用包装器，获取设备信息
    if (!callWrapper(CudaLib::deviceInfo(m_ctx, m_data.thread.blocks(), m_data.thread.threads(), m_data.algorithm, m_data.thread.datasetHost()))) {
        return false;
    }
    # 调用 callWrapper 函数，传入 CudaLib::deviceInit(m_ctx) 的返回值作为参数，并返回结果
    return callWrapper(CudaLib::deviceInit(m_ctx));
# 设置 CUDA 运行器的参数，包括作业高度和目标值
bool xmrig::CudaBaseRunner::set(const Job &job, uint8_t *blob)
{
    # 设置作业高度
    m_height = job.height();
    # 设置作业目标值
    m_target = job.target();

    # 调用 CUDA 库的 setJob 方法，设置作业参数
    return callWrapper(CudaLib::setJob(m_ctx, blob, job.size(), job.algorithm()));
}

# 返回 CUDA 运行器的强度
size_t xmrig::CudaBaseRunner::intensity() const
{
    # 返回线程数乘以块数，作为强度
    return m_data.thread.threads() * m_data.thread.blocks();
}

# 调用 CUDA 运行器的包装方法
bool xmrig::CudaBaseRunner::callWrapper(bool result) const
{
    # 如果结果为假
    if (!result) {
        # 获取 CUDA 库的最后一个错误信息
        const char *error = CudaLib::lastError(m_ctx);
        # 如果存在错误信息
        if (error) {
            # 记录错误信息到日志中
            LOG_ERR("%s" RED_S " thread " RED_BOLD("#%zu") RED_S " failed with error " RED_BOLD("%s"), cuda_tag(), m_threadId, error);
        }
    }

    # 返回结果
    return result;
}
```