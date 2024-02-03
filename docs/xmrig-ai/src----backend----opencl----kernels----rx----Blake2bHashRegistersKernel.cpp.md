# `xmrig\src\backend\opencl\kernels\rx\Blake2bHashRegistersKernel.cpp`

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
 * 本程序是自由软件：您可以根据 GNU 通用公共许可证的条款重新分发或修改它，
 * 该许可证由自由软件基金会发布，无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是基于希望它有用的前提下分发的，但没有任何担保；甚至没有适用于特定目的的隐含担保。
 * 有关更多详细信息，请参阅 GNU 通用公共许可证。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本。
 * 如果没有，请参阅 <http://www.gnu.org/licenses/>。
 */

#include "backend/opencl/kernels/rx/Blake2bHashRegistersKernel.h"
#include "backend/opencl/wrappers/OclLib.h"

// 将任务加入到 OpenCL 命令队列中
void xmrig::Blake2bHashRegistersKernel::enqueue(cl_command_queue queue, size_t threads)
{
    const size_t gthreads        = threads;  // 设置全局线程数
    static const size_t lthreads = 64;      // 设置本地线程数为 64

    enqueueNDRange(queue, 1, nullptr, &gthreads, &lthreads);  // 将任务加入到命令队列中，使用指定的全局和本地线程数
}

// 设置内核参数
void xmrig::Blake2bHashRegistersKernel::setArgs(cl_mem out, cl_mem in, uint32_t inStrideBytes)
{
    setArg(0, sizeof(cl_mem), &out);  // 设置第一个参数为输出内存对象
    setArg(1, sizeof(cl_mem), &in);   // 设置第二个参数为输入内存对象
}
    # 设置参数2的值为uint32_t类型的大小，并将inStrideBytes的地址传递给该参数
    setArg(2, sizeof(uint32_t), &inStrideBytes);
# 闭合函数定义的大括号，表示函数定义结束
```