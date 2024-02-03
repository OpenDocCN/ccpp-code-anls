# `xmrig\src\backend\opencl\runners\OclKawPowRunner.h`

```cpp
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   它可以是许可证的第3版，也可以是（在您的选择下）任何以后的版本。
 *
 *   本程序是在希望它有用的情况下分发的，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。有关更多详细信息，请参见
 *   GNU通用公共许可证。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_OCLKAWPOWRUNNER_H
#define XMRIG_OCLKAWPOWRUNNER_H


#include "backend/opencl/runners/OclBaseRunner.h"
#include "crypto/kawpow/KPCache.h"

#include <mutex>

namespace xmrig {


class KawPow_CalculateDAGKernel;


class OclKawPowRunner : public OclBaseRunner
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(OclKawPowRunner)

    OclKawPowRunner(size_t index, const OclLaunchData &data);
    ~OclKawPowRunner() override;

protected:
    void run(uint32_t nonce, uint32_t *hashOutput) override;
    void set(const Job &job, uint8_t *blob) override;
    void build() override;
    void init() override;
    void jobEarlyNotification(const Job& job) override;
    uint32_t processedHashes() const override { return m_intensity - m_skippedHashes; }

private:
    uint8_t* m_blob = nullptr;  // 初始化为nullptr的uint8_t指针
    uint32_t m_skippedHashes = 0;  // 初始化为0的uint32_t

    uint32_t m_blockHeight = 0;  // 初始化为0的uint32_t
    uint32_t m_epoch = 0xFFFFFFFFUL;  // 初始化为0xFFFFFFFFUL的uint32_t

    cl_mem m_lightCache = nullptr;  // 初始化为nullptr的cl_mem
    size_t m_lightCacheSize = 0;  // 初始化为0的size_t
    size_t m_lightCacheCapacity = 0;  // 初始化为0的size_t

    cl_mem m_dag = nullptr;  // 初始化为nullptr的cl_mem
    size_t m_dagCapacity = 0;  // 初始化为0的size_t

    KawPow_CalculateDAGKernel* m_calculateDagKernel = nullptr;  // 初始化为nullptr的KawPow_CalculateDAGKernel指针
    # 定义一个指向 OpenCL 内核的指针变量，并初始化为 nullptr
    cl_kernel m_searchKernel = nullptr;
    
    # 定义一个表示工作组大小的变量，并初始化为 256
    size_t m_workGroupSize = 256;
    
    # 定义一个表示 DAG 工作组大小的变量，并初始化为 64
    size_t m_dagWorkGroupSize = 64;
    
    # 定义一个指向 OpenCL 命令队列的指针变量，并初始化为 nullptr
    cl_command_queue m_controlQueue = nullptr;
    
    # 定义一个指向 OpenCL 内存对象的指针变量，并初始化为 nullptr
    cl_mem m_stop = nullptr;
}; 
// 结束了一个命名空间的定义

} /* namespace xmrig */
// 命名空间结束

#endif // XMRIG_OCLKAWPOWRUNNER_H
// 结束了对头文件的引用
```