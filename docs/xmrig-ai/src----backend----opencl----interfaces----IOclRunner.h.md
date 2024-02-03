# `xmrig\src\backend\opencl\interfaces\IOclRunner.h`

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
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_IOCLRUNNER_H
#define XMRIG_IOCLRUNNER_H


#include "base/tools/Object.h"


#include <cstdint>


using cl_context = struct _cl_context *;


namespace xmrig {


class Algorithm;
class Job;
class OclLaunchData;


class IOclRunner
{
public:
    XMRIG_DISABLE_COPY_MOVE(IOclRunner)

    IOclRunner()          = default;
    virtual ~IOclRunner() = default;

    virtual cl_context ctx() const                          = 0;  // 返回 OpenCL 上下文
    virtual const Algorithm &algorithm() const              = 0;  // 返回算法
    virtual const char *buildOptions() const                = 0;  // 返回构建选项
    virtual const char *deviceKey() const                   = 0;  // 返回设备密钥
    virtual const char *source() const                      = 0;  // 返回源代码
    virtual const OclLaunchData &data() const               = 0;  // 返回 OpenCL 启动数据
    # 返回虚拟设备的强度
    virtual size_t intensity() const                        = 0;
    # 返回线程的ID
    virtual size_t threadId() const                         = 0;
    # 返回轮大小
    virtual uint32_t roundSize() const                      = 0;
    # 返回已处理的哈希数量
    virtual uint32_t processedHashes() const                = 0;
    # 返回设备索引
    virtual uint32_t deviceIndex() const                    = 0;
    # 构建函数，无返回值
    virtual void build()                                    = 0;
    # 初始化函数，无返回值
    virtual void init()                                     = 0;
    # 运行函数，接受nonce和hashOutput参数，无返回值
    virtual void run(uint32_t nonce, uint32_t *hashOutput)  = 0;
    # 设置函数，接受Job和blob参数，无返回值
    virtual void set(const Job &job, uint8_t *blob)         = 0;
    # 作业提前通知函数，接受Job参数，无返回值
    virtual void jobEarlyNotification(const Job&)           = 0;
// 声明一个虚函数，返回一个大小的值，该函数在派生类中实现
protected:
    virtual size_t bufferSize() const                       = 0;
};


} /* namespace xmrig */


#endif // XMRIG_IOCLRUNNER_H
```