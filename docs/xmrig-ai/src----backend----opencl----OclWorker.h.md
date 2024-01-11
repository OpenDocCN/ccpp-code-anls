# `xmrig\src\backend\opencl\OclWorker.h`

```
/* XMRig
 * 版权所有（c）2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的，
 *   但没有任何保证；甚至没有适销性或特定用途的暗示保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_OCLWORKER_H
#define XMRIG_OCLWORKER_H


#include "backend/common/GpuWorker.h"
#include "backend/common/WorkerJob.h"
#include "backend/opencl/OclLaunchData.h"
#include "base/tools/Object.h"
#include "net/JobResult.h"


namespace xmrig {


class IOclRunner;
class Job;


class OclWorker : public GpuWorker
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(OclWorker)

    // 构造函数，初始化 OclWorker 对象
    OclWorker(size_t id, const OclLaunchData &data);

    // 析构函数
    ~OclWorker() override;

    // 作业提前通知
    void jobEarlyNotification(const Job &job) override;

    // 静态成员变量，表示是否准备就绪
    static std::atomic<bool> ready;

protected:
    // 自我测试
    bool selfTest() override;
    // 强度
    size_t intensity() const override;
    // 启动
    void start() override;

private:
    // 处理作业
    bool consumeJob();
    // 存储统计数据
    void storeStats(uint64_t ts);

    // 成员变量
    const Algorithm m_algorithm;
    const Miner *m_miner;
    IOclRunner *m_runner = nullptr;
    OclSharedData &m_sharedData;
    WorkerJob<1> m_job;
};


} // namespace xmrig


#endif /* XMRIG_OCLWORKER_H */
```