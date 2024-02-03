# `xmrig\src\backend\opencl\runners\OclRxVmRunner.h`

```cpp
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。有关更多详细信息，请参见
 *   GNU通用公共许可证。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_OCLRXVMRUNNER_H
#define XMRIG_OCLRXVMRUNNER_H


#include "backend/opencl/runners/OclRxBaseRunner.h"


namespace xmrig {


class ExecuteVmKernel;
class InitVmKernel;


class OclRxVmRunner : public OclRxBaseRunner
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(OclRxVmRunner)

    OclRxVmRunner(size_t index, const OclLaunchData &data);
    ~OclRxVmRunner() override;

protected:
    size_t bufferSize() const override;
    void build() override;
    void execute(uint32_t iteration) override;
    void init() override;

private:
    cl_mem m_vm_states              = nullptr;  // 虚拟机状态的内存对象
    ExecuteVmKernel *m_execute_vm   = nullptr;  // 执行虚拟机的内核对象
    InitVmKernel *m_init_vm         = nullptr;  // 初始化虚拟机的内核对象
};


} /* namespace xmrig */


#endif // XMRIG_OCLRXVMRUNNER_H
```