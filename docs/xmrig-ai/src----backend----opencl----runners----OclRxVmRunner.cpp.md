# `xmrig\src\backend\opencl\runners\OclRxVmRunner.cpp`

```
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新分发或修改
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的，
 *   但没有任何保证；甚至没有暗示的保证适用于特定用途。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "backend/opencl/runners/OclRxVmRunner.h"
#include "backend/opencl/kernels/rx/Blake2bHashRegistersKernel.h"
#include "backend/opencl/kernels/rx/ExecuteVmKernel.h"
#include "backend/opencl/kernels/rx/HashAesKernel.h"
#include "backend/opencl/kernels/rx/InitVmKernel.h"
#include "backend/opencl/OclLaunchData.h"
#include "backend/opencl/wrappers/OclLib.h"
#include "crypto/rx/RxAlgo.h"


#include <algorithm>


// 构造函数，初始化 OclRxVmRunner 对象
xmrig::OclRxVmRunner::OclRxVmRunner(size_t index, const OclLaunchData &data) : OclRxBaseRunner(index, data)
{
}


// 析构函数，释放内存
xmrig::OclRxVmRunner::~OclRxVmRunner()
{
    delete m_init_vm;
    delete m_execute_vm;

    OclLib::release(m_vm_states);
}


// 返回缓冲区大小
size_t xmrig::OclRxVmRunner::bufferSize() const
{
    return OclRxBaseRunner::bufferSize() + (align(2560 * m_intensity));
}


// 构建函数
void xmrig::OclRxVmRunner::build()
{
    OclRxBaseRunner::build();

    // 计算哈希步长字节数
    const uint32_t hashStrideBytes = RxAlgo::programSize(m_algorithm) * 8;

    // 设置参数
    m_hashAes1Rx4->setArgs(m_scratchpads, m_vm_states, hashStrideBytes, m_intensity);
    m_blake2b_hash_registers_32->setArgs(m_hashes, m_vm_states, hashStrideBytes);
    # 设置 m_hashes, m_vm_states, hashStrideBytes 为参数，传递给 m_blake2b_hash_registers_64 对象
    m_blake2b_hash_registers_64->setArgs(m_hashes, m_vm_states, hashStrideBytes);

    # 使用 m_program 创建 InitVmKernel 对象，并设置 m_entropy, m_vm_states, m_rounding 为参数
    m_init_vm = new InitVmKernel(m_program);
    m_init_vm->setArgs(m_entropy, m_vm_states, m_rounding);

    # 使用 m_program 创建 ExecuteVmKernel 对象，并设置 m_vm_states, m_rounding, m_scratchpads, m_dataset, m_intensity 为参数
    m_execute_vm = new ExecuteVmKernel(m_program);
    m_execute_vm->setArgs(m_vm_states, m_rounding, m_scratchpads, m_dataset, m_intensity);
# 执行函数，接受一个迭代次数作为参数
void xmrig::OclRxVmRunner::execute(uint32_t iteration)
{
    # 计算bfactor，取data().thread.bfactor()和8的最小值
    const uint32_t bfactor        = std::min(data().thread.bfactor(), 8U);
    # 计算num_iterations，使用RxAlgo::programIterations(m_algorithm)右移bfactor位得到
    const uint32_t num_iterations = RxAlgo::programIterations(m_algorithm) >> bfactor;

    # 将初始化虚拟机的任务加入队列，使用m_intensity和iteration作为参数
    m_init_vm->enqueue(m_queue, m_intensity, iteration);

    # 设置执行虚拟机的迭代次数
    m_execute_vm->setIterations(num_iterations);

    # 循环执行虚拟机任务
    for (int j = 0, n = 1 << bfactor; j < n; ++j) {
        # 如果j等于n-1，设置执行虚拟机的最后一次任务标志为1
        if (j == n - 1) {
            m_execute_vm->setLast(1);
        }

        # 将执行虚拟机的任务加入队列，使用m_intensity和m_worksize作为参数
        m_execute_vm->enqueue(m_queue, m_intensity, m_worksize);

        # 如果j等于0，设置执行虚拟机的第一次任务标志为0
        if (j == 0) {
            m_execute_vm->setFirst(0);
        }
    }
}

# 初始化函数
void xmrig::OclRxVmRunner::init()
{
    # 调用父类的初始化函数
    OclRxBaseRunner::init();

    # 创建大小为2560*m_intensity的子缓冲区，类型为CL_MEM_READ_WRITE，赋值给m_vm_states
    m_vm_states = createSubBuffer(CL_MEM_READ_WRITE, 2560 * m_intensity);
}
```