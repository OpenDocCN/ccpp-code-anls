# `xmrig\src\backend\cuda\runners\CudaBaseRunner.h`

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
 * 本程序是自由软件：您可以重新发布和/或修改
 * 根据GNU通用公共许可证的条款发布
 * 由自由软件基金会发布，无论是许可证的第3版
 * （在您的选择）任何后续版本。
 *
 * 本程序是希望它有用
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_CUDABASERUNNER_H
#define XMRIG_CUDABASERUNNER_H

// 包含ICudaRunner接口
#include "backend/cuda/interfaces/ICudaRunner.h"

// 使用nvid_ctx结构
using nvid_ctx = struct nvid_ctx;

// 命名空间xmrig
namespace xmrig {

// CudaLaunchData类的前向声明
class CudaLaunchData;

// CudaBaseRunner类继承自ICudaRunner接口
class CudaBaseRunner : public ICudaRunner
{
public:
    // 禁用默认的拷贝和移动构造函数
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(CudaBaseRunner)

    // 构造函数，接受id和CudaLaunchData对象的引用
    CudaBaseRunner(size_t id, const CudaLaunchData &data);
    // 虚析构函数
    ~CudaBaseRunner() override;

protected:
    // 初始化函数，覆盖自ICudaRunner接口
    bool init() override;
    // 设置函数，覆盖自ICudaRunner接口，接受Job对象和blob指针
    bool set(const Job &job, uint8_t *blob) override;
    // 强度函数，覆盖自ICudaRunner接口
    size_t intensity() const override;
    // roundSize函数，覆盖自ICudaRunner接口，返回强度值
    size_t roundSize() const override { return intensity(); }
    // processedHashes函数，覆盖自ICudaRunner接口，返回强度值
    size_t processedHashes() const override { return intensity(); }
    // jobEarlyNotification函数，覆盖自ICudaRunner接口，接受Job对象
    void jobEarlyNotification(const Job&) override {}

protected:
    // 调用包装器函数，接受布尔值结果
    bool callWrapper(bool result) const;
    # 声明一个常量引用类型的CudaLaunchData对象m_data
    const CudaLaunchData &m_data;
    # 声明一个常量大小的线程ID变量m_threadId
    const size_t m_threadId;
    # 声明一个指向nvid_ctx类型的指针变量m_ctx，并初始化为nullptr
    nvid_ctx *m_ctx     = nullptr;
    # 声明一个64位无符号整数变量m_height，并初始化为0
    uint64_t m_height   = 0;
    # 声明一个64位无符号整数变量m_target，并初始化为0
    uint64_t m_target   = 0;
}; 
// 结束了 xmrig 命名空间的定义

} /* namespace xmrig */
// 回到 xmrig 命名空间

#endif // XMRIG_CUDABASERUNNER_H
// 结束了 XMRIG_CUDABASERUNNER_H 的定义
```