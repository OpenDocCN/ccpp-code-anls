# `xmrig\src\backend\cuda\CudaBackend.h`

```cpp
/* XMRig
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据 GNU 通用公共许可证的条款发布，由
 *   自由软件基金会发布的版本3或
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_CUDABACKEND_H
#define XMRIG_CUDABACKEND_H


#include <utility>


#include "backend/common/interfaces/IBackend.h"
#include "base/tools/Object.h"


namespace xmrig {


class Controller;
class CudaBackendPrivate;
class Miner;


class CudaBackend : public IBackend
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(CudaBackend)

    CudaBackend(Controller *controller);

    ~CudaBackend() override;

protected:
    bool isEnabled() const override;
    bool isEnabled(const Algorithm &algorithm) const override;
    const Hashrate *hashrate() const override;
    const String &profileName() const override;
    const String &type() const override;
    void execCommand(char command) override;
    void prepare(const Job &nextJob) override;
    void printHashrate(bool details) override;
    void printHealth() override;
    void setJob(const Job &job) override;
    void start(IWorker *worker, bool ready) override;
    void stop() override;
    bool tick(uint64_t ticks) override;

#   ifdef XMRIG_FEATURE_API
    rapidjson::Value toJSON(rapidjson::Document &doc) const override;
    void handleRequest(IApiRequest &request) override;
#   endif
#   ifdef XMRIG_FEATURE_BENCHMARK
    // 如果定义了 XMRIG_FEATURE_BENCHMARK，则实现 benchmark() 方法返回空指针
    inline Benchmark *benchmark() const override        { return nullptr; }
    // 如果定义了 XMRIG_FEATURE_BENCHMARK，则实现 printBenchProgress() 方法为空
    inline void printBenchProgress() const override     {}
#   endif

private:
    // 私有成员变量，指向 CudaBackendPrivate 类的指针
    CudaBackendPrivate *d_ptr;
};


} /* namespace xmrig */


#endif /* XMRIG_CUDABACKEND_H */
```