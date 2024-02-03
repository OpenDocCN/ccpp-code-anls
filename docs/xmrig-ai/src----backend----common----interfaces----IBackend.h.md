# `xmrig\src\backend\common\interfaces\IBackend.h`

```cpp
/*
 * XMRig
 * 版权所有（c）2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款，
 * 无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用的分发，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。有关更多详细信息，请参见
 * GNU通用公共许可证。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_IBACKEND_H
#define XMRIG_IBACKEND_H


#include "3rdparty/rapidjson/fwd.h"
#include "base/tools/Object.h"


#include <cstdint>


namespace xmrig {


class Algorithm;
class Benchmark;
class Hashrate;
class IApiRequest;
class IWorker;
class Job;
class String;


class IBackend
{
public:
    XMRIG_DISABLE_COPY_MOVE(IBackend)

    // 默认构造函数
    IBackend()          = default;
    // 虚析构函数
    virtual ~IBackend() = default;

    // 检查后端是否启用
    virtual bool isEnabled() const                                      = 0;
    // 检查特定算法的后端是否启用
    virtual bool isEnabled(const Algorithm &algorithm) const            = 0;
    // 执行时钟周期
    virtual bool tick(uint64_t ticks)                                   = 0;
    // 获取哈希率
    virtual const Hashrate *hashrate() const                            = 0;
    // 获取配置文件名称
    virtual const String &profileName() const                           = 0;
    // 获取类型
    virtual const String &type() const                                  = 0;
    // 执行命令
    virtual void execCommand(char command)                              = 0;
    // 准备工作
    virtual void prepare(const Job &nextJob)                            = 0;
    // 打印哈希率
    virtual void printHashrate(bool details)                            = 0;
    # 声明一个虚拟函数，用于打印健康状态
    virtual void printHealth() = 0;
    # 声明一个虚拟函数，用于设置工作
    virtual void setJob(const Job &job) = 0;
    # 声明一个虚拟函数，用于启动工作，传入工作者对象和是否准备就绪的标志
    virtual void start(IWorker *worker, bool ready) = 0;
    # 声明一个虚拟函数，用于停止工作
    virtual void stop() = 0;
#   ifdef XMRIG_FEATURE_API
    # 如果定义了 XMRIG_FEATURE_API，则执行以下代码

    # 将对象转换为 JSON 格式
    virtual rapidjson::Value toJSON(rapidjson::Document &doc) const     = 0;
    # 处理 API 请求
    virtual void handleRequest(IApiRequest &request)                    = 0;
#   endif
    # 结束条件编译指令

#   ifdef XMRIG_FEATURE_BENCHMARK
    # 如果定义了 XMRIG_FEATURE_BENCHMARK，则执行以下代码

    # 获取基准测试对象
    virtual Benchmark *benchmark() const                                = 0;
    # 打印基准测试进度
    virtual void printBenchProgress() const                             = 0;
#   endif
    # 结束条件编译指令

};
# 结束命名空间 xmrig

#endif // XMRIG_IBACKEND_H
```