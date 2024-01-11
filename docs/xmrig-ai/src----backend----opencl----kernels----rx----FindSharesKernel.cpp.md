# `xmrig\src\backend\opencl\kernels\rx\FindSharesKernel.cpp`

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
 * 本程序是自由软件：您可以根据 GNU 通用公共许可证的条款重新分发或修改它，无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是基于希望它有用而分发的，但没有任何担保；甚至没有适销性或特定用途的暗示担保。有关更多详细信息，请参见 GNU 通用公共许可证。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#include "backend/opencl/kernels/rx/FindSharesKernel.h"
#include "backend/opencl/wrappers/OclLib.h"

// 将任务加入到命令队列中
void xmrig::FindSharesKernel::enqueue(cl_command_queue queue, size_t threads)
{
    const size_t gthreads        = threads;
    static const size_t lthreads = 64;

    // 将任务加入到命令队列中，指定全局工作项数和局部工作项数
    enqueueNDRange(queue, 1, nullptr, &gthreads, &lthreads);
}

// 设置内核参数
void xmrig::FindSharesKernel::setArgs(cl_mem hashes, cl_mem shares)
{
    // 设置内核参数，指定参数索引、参数大小和参数值
    setArg(0, sizeof(cl_mem), &hashes);
    setArg(3, sizeof(cl_mem), &shares);
}

// 设置目标值
void xmrig::FindSharesKernel::setTarget(uint64_t target)
{
    // 设置内核参数，指定参数索引、参数大小和参数值
    setArg(1, sizeof(uint64_t), &target);
}
# 设置工作量证明的随机数
void xmrig::FindSharesKernel::setNonce(uint32_t nonce)
{
    # 设置内核函数参数，参数索引为2，参数大小为uint32_t类型大小，参数值为nonce的地址
    setArg(2, sizeof(uint32_t), &nonce);
}
```