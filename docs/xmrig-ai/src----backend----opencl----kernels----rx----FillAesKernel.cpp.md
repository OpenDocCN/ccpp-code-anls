# `xmrig\src\backend\opencl\kernels\rx\FillAesKernel.cpp`

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
 * 本程序是自由软件：您可以根据 GNU 通用公共许可证的条款重新分发或修改它，无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是基于希望它有用而分发的，但没有任何担保；甚至没有适销性或特定用途的暗示担保。有关更多详细信息，请参见 GNU 通用公共许可证。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#include "backend/opencl/kernels/rx/FillAesKernel.h"
#include "backend/opencl/wrappers/OclLib.h"

// 将任务加入到 OpenCL 命令队列中
void xmrig::FillAesKernel::enqueue(cl_command_queue queue, size_t threads)
{
    // 计算全局线程数
    const size_t gthreads        = threads * 4;
    // 定义局部线程数
    static const size_t lthreads = 64;

    // 将任务加入到命令队列中
    enqueueNDRange(queue, 1, nullptr, &gthreads, &lthreads);
}

// 设置参数
void xmrig::FillAesKernel::setArgs(cl_mem state, cl_mem out, uint32_t batch_size, uint32_t rx_version)
{
    // 设置参数0
    setArg(0, sizeof(cl_mem), &state);
    // 设置参数1
    setArg(1, sizeof(cl_mem), &out);
    # 设置参数2，指定参数类型为uint32_t，传入batch_size的地址
    setArg(2, sizeof(uint32_t), &batch_size);
    # 设置参数3，指定参数类型为uint32_t，传入rx_version的地址
    setArg(3, sizeof(uint32_t), &rx_version);
# 闭合前面的函数定义
```