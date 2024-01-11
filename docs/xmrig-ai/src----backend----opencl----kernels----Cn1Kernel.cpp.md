# `xmrig\src\backend\opencl\kernels\Cn1Kernel.cpp`

```
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
 * 根据 GNU 通用公共许可证的条款发布
 * 由自由软件基金会发布的许可证的第3版或
 * （根据您的选择）任何以后的版本。
 *
 * 本程序是希望它有用
 * 但没有任何保证；甚至没有暗示的保证
 * 商品性或适用于特定目的。请参阅
 * GNU 通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include <string>


#include "backend/opencl/kernels/Cn1Kernel.h"
#include "backend/opencl/wrappers/OclLib.h"

// 构造函数，初始化 cn1 内核
xmrig::Cn1Kernel::Cn1Kernel(cl_program program)
    : OclKernel(program, "cn1")
{
}

// 构造函数，初始化 cn1 内核，带有高度参数
xmrig::Cn1Kernel::Cn1Kernel(cl_program program, uint64_t height)
    : OclKernel(program, ("cn1_" + std::to_string(height)).c_str())
{
}

// 将任务加入队列
void xmrig::Cn1Kernel::enqueue(cl_command_queue queue, uint32_t nonce, size_t threads, size_t worksize)
{
    const size_t offset   = nonce;
    const size_t gthreads = threads;
    const size_t lthreads = worksize;

    // 将任务加入队列
    enqueueNDRange(queue, 1, &offset, &gthreads, &lthreads);
}

// __kernel void cn1(__global ulong *input, __global uint4 *Scratchpad, __global ulong *states, uint Threads)
# 设置 OpenCL 内核函数的参数
void xmrig::Cn1Kernel::setArgs(cl_mem input, cl_mem scratchpads, cl_mem states, uint32_t threads)
{
    # 设置第一个参数，输入内存对象
    setArg(0, sizeof(cl_mem), &input);
    # 设置第二个参数，临时内存对象
    setArg(1, sizeof(cl_mem), &scratchpads);
    # 设置第三个参数，状态内存对象
    setArg(2, sizeof(cl_mem), &states);
    # 设置第四个参数，线程数
    setArg(3, sizeof(uint32_t), &threads);
}
```