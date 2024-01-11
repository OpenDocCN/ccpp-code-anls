# `xmrig\src\backend\opencl\OclBackend.h`

```
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据GNU通用公共许可证的条款发布，由
 *   自由软件基金会发布的许可证的第3版，或者
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_OCLBACKEND_H
#define XMRIG_OCLBACKEND_H


#include <utility>


#include "backend/common/interfaces/IBackend.h"
#include "base/tools/Object.h"


namespace xmrig {


class Controller;
class OclBackendPrivate;
class Miner;


class OclBackend : public IBackend
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(OclBackend)

    OclBackend(Controller *controller);

    ~OclBackend() override;

protected:
    // 检查OpenCL后端是否启用
    bool isEnabled() const override;
    // 检查特定算法的OpenCL后端是否启用
    bool isEnabled(const Algorithm &algorithm) const override;
    // 获取哈希率
    const Hashrate *hashrate() const override;
    // 获取配置文件名称
    const String &profileName() const override;
    // 获取类型
    const String &type() const override;
    // 执行命令
    void execCommand(char command) override;
    // 准备工作
    void prepare(const Job &nextJob) override;
    // 打印哈希率
    void printHashrate(bool details) override;
    // 打印健康状态
    void printHealth() override;
    // 设置工作
    void setJob(const Job &job) override;
    // 启动
    void start(IWorker *worker, bool ready) override;
    // 停止
    void stop() override;
    // 定时器
    bool tick(uint64_t ticks) override;

#   ifdef XMRIG_FEATURE_API
    // 转换为JSON格式
    rapidjson::Value toJSON(rapidjson::Document &doc) const override;
    // 处理API请求
    void handleRequest(IApiRequest &request) override;
#   endif
// 如果定义了 XMRIG_FEATURE_BENCHMARK，则实现 benchmark() 方法返回空指针
#ifdef XMRIG_FEATURE_BENCHMARK
    inline Benchmark *benchmark() const override        { return nullptr; }
    // 如果定义了 XMRIG_FEATURE_BENCHMARK，则实现 printBenchProgress() 方法为空
    inline void printBenchProgress() const override     {}
#endif

// 私有变量部分开始
private:
    // 指向 OclBackendPrivate 类的指针
    OclBackendPrivate *d_ptr;
};

// 命名空间结束
} /* namespace xmrig */

// 结束条件编译指令，关闭 XMRIG_OCLBACKEND_H 头文件的定义
#endif /* XMRIG_OCLBACKEND_H */
```