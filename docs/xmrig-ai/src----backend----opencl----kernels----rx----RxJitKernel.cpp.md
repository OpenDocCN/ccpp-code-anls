# `xmrig\src\backend\opencl\kernels\rx\RxJitKernel.cpp`

```cpp
/* XMRig
 * 版权所有 2010      Jeff Garzik <jgarzik@pobox.com>
 * 版权所有 2012-2014 pooler      <pooler@litecoinpool.org>
 * 版权所有 2014      Lucas Jones <https://github.com/lucasjones>
 * 版权所有 2014-2016 Wolf9466    <https://github.com/OhGodAPet>
 * 版权所有 2016      Jay D Dee   <jayddee246@gmail.com>
 * 版权所有 2017-2018 XMR-Stak    <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * 版权所有 2018-2019 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2019 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据 GNU 通用公共许可证的条款发布，由
 *   自由软件基金会发布的许可证的第3版或
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用：
 *   但没有任何保证；甚至没有暗示的保证
 *   商品性或适用于特定目的。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */


#include "backend/opencl/kernels/rx/RxJitKernel.h"
#include "backend/opencl/wrappers/OclLib.h"


void xmrig::RxJitKernel::enqueue(cl_command_queue queue, size_t threads, uint32_t iteration)
{
    setArg(6, sizeof(uint32_t), &iteration);  // 设置内核参数，用于迭代

    const size_t gthreads        = threads * 32;  // 计算全局工作组大小
    static const size_t lthreads = 64;  // 设置本地工作组大小

    enqueueNDRange(queue, 1, nullptr, &gthreads, &lthreads);  // 将内核加入到命令队列中
}


// __kernel void randomx_jit(__global ulong* entropy, __global ulong* registers, __global uint2* intermediate_programs, __global uint* programs, uint batch_size, __global uint32_t* rounding, uint32_t iteration)
void xmrig::RxJitKernel::setArgs(cl_mem entropy, cl_mem registers, cl_mem intermediate_programs, cl_mem programs, uint32_t batch_size, cl_mem rounding)
{
    # 设置第一个参数，传入entropy的内存大小和地址
    setArg(0, sizeof(cl_mem), &entropy);
    # 设置第二个参数，传入registers的内存大小和地址
    setArg(1, sizeof(cl_mem), &registers);
    # 设置第三个参数，传入intermediate_programs的内存大小和地址
    setArg(2, sizeof(cl_mem), &intermediate_programs);
    # 设置第四个参数，传入programs的内存大小和地址
    setArg(3, sizeof(cl_mem), &programs);
    # 设置第五个参数，传入batch_size的大小
    setArg(4, sizeof(uint32_t), &batch_size);
    # 设置第六个参数，传入rounding的内存大小和地址
    setArg(5, sizeof(cl_mem), &rounding);
# 闭合前面的函数定义
```