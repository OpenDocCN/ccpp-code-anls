# `xmrig\src\backend\cpu\CpuBackend.h`

```
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的，
 *   但没有任何保证；甚至没有暗示的保证适销性或特定用途的保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_CPUBACKEND_H
#define XMRIG_CPUBACKEND_H


#include "backend/common/interfaces/IBackend.h"
#include "base/tools/Object.h"


#include <utility>


namespace xmrig {


class Controller;
class CpuBackendPrivate;
class Miner;


class CpuBackend : public IBackend
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(CpuBackend)

    CpuBackend(Controller *controller);
    ~CpuBackend() override;

protected:
    inline void execCommand(char) override {}

    bool isEnabled() const override;
    bool isEnabled(const Algorithm &algorithm) const override;
    bool tick(uint64_t ticks) override;
    const Hashrate *hashrate() const override;
    const String &profileName() const override;
    const String &type() const override;
    void prepare(const Job &nextJob) override;
    void printHashrate(bool details) override;
    void printHealth() override;
    void setJob(const Job &job) override;
    void start(IWorker *worker, bool ready) override;
    void stop() override;

#   ifdef XMRIG_FEATURE_API
    rapidjson::Value toJSON(rapidjson::Document &doc) const override;
    void handleRequest(IApiRequest &request) override;
#   endif
// 如果定义了 XMRIG_FEATURE_BENCHMARK，则声明 Benchmark 函数并覆盖
#ifdef XMRIG_FEATURE_BENCHMARK
    Benchmark *benchmark() const override;
    // 声明打印基准测试进度的函数并覆盖
    void printBenchProgress() const override;
#endif

// 私有成员变量
private:
    // 指向 CpuBackendPrivate 类的指针
    CpuBackendPrivate *d_ptr;
};

// 命名空间 xmrig 结束
} /* namespace xmrig */

// 结束条件编译
#endif /* XMRIG_CPUBACKEND_H */
```