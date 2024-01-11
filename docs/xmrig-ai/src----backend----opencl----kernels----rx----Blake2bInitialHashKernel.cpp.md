# `xmrig\src\backend\opencl\kernels\rx\Blake2bInitialHashKernel.cpp`

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
 * 根据 GNU 通用公共许可证的条款发布，
 * 由自由软件基金会发布的版本 3 或
 * （根据您的选择）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU 通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "backend/opencl/kernels/rx/Blake2bInitialHashKernel.h"
#include "backend/opencl/wrappers/OclLib.h"

// 将任务加入队列
void xmrig::Blake2bInitialHashKernel::enqueue(cl_command_queue queue, size_t threads)
{
    const size_t gthreads        = threads;  // 设置全局线程数
    static const size_t lthreads = 64;       // 设置本地线程数

    enqueueNDRange(queue, 1, nullptr, &gthreads, &lthreads);  // 将任务加入队列
}

// 设置参数
void xmrig::Blake2bInitialHashKernel::setArgs(cl_mem out, cl_mem blockTemplate)
{
    setArg(0, sizeof(cl_mem), &out);          // 设置参数0
    setArg(1, sizeof(cl_mem), &blockTemplate); // 设置参数1
}

// 设置块大小
void xmrig::Blake2bInitialHashKernel::setBlobSize(size_t size)
{
    const uint32_t s = size;  // 设置块大小
}
    # 设置参数2的值为一个32位无符号整数的大小，并将其存储在变量s中
    setArg(2, sizeof(uint32_t), &s);
# 设置 Blake2bInitialHashKernel 类的 nonce 属性
void xmrig::Blake2bInitialHashKernel::setNonce(uint32_t nonce)
{
    # 调用 setArg 方法设置第四个参数为 nonce 的大小和值
    setArg(3, sizeof(uint32_t), &nonce);
}
```