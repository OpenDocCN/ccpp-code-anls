# `xmrig\src\backend\opencl\kernels\rx\ExecuteVmKernel.h`

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
 * 根据 GNU 通用公共许可证的条款发布，
 * 由自由软件基金会发布的版本 3 或
 * （根据您的选择）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证。
 * 有关更多详细信息，请参见
 * GNU 通用公共许可证。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_EXECUTEVMKERNEL_H
#define XMRIG_EXECUTEVMKERNEL_H

#include "backend/opencl/wrappers/OclKernel.h"

namespace xmrig {

class ExecuteVmKernel : public OclKernel
{
public:
    inline ExecuteVmKernel(cl_program program) : OclKernel(program, "execute_vm") {}  // 使用给定的程序和内核名称初始化 ExecuteVmKernel 对象

    void enqueue(cl_command_queue queue, size_t threads, size_t worksize);  // 将内核排入命令队列，指定线程数和工作大小
    void setArgs(cl_mem vm_states, cl_mem rounding, cl_mem scratchpads, cl_mem dataset_ptr, uint32_t batch_size);  // 设置内核参数
    void setFirst(uint32_t first);  // 设置第一个参数
    void setIterations(uint32_t num_iterations);  // 设置迭代次数
    void setLast(uint32_t last);  // 设置最后一个参数
};

} // namespace xmrig

#endif /* XMRIG_EXECUTEVMKERNEL_H */
```