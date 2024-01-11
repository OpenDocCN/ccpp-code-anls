# `xmrig\src\backend\opencl\kernels\rx\HashAesKernel.cpp`

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
 * 由自由软件基金会发布，无论是许可证的第3版，
 * 还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。更多细节请参见
 * GNU 通用公共许可证。
 *
 * 您应该已经收到 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "backend/opencl/kernels/rx/HashAesKernel.h"
#include "backend/opencl/wrappers/OclLib.h"

// 将任务加入队列
void xmrig::HashAesKernel::enqueue(cl_command_queue queue, size_t threads)
{
    // 计算全局线程数
    const size_t gthreads        = threads * 4;
    // 定义局部线程数
    static const size_t lthreads = 64;

    // 将任务加入队列
    enqueueNDRange(queue, 1, nullptr, &gthreads, &lthreads);
}

// 设置参数
void xmrig::HashAesKernel::setArgs(cl_mem input, cl_mem hash, uint32_t hashStrideBytes, uint32_t batch_size)
{
    // 定义哈希偏移字节数
    const uint32_t hashOffsetBytes = 192;

    // 设置参数
    setArg(0, sizeof(cl_mem), &input);
    setArg(1, sizeof(cl_mem), &hash);
    setArg(2, sizeof(uint32_t), &hashOffsetBytes);
}
    # 设置第三个参数，指定为 uint32_t 类型的大小，传入 hashStrideBytes 的地址
    setArg(3, sizeof(uint32_t), &hashStrideBytes);
    # 设置第四个参数，指定为 uint32_t 类型的大小，传入 batch_size 的地址
    setArg(4, sizeof(uint32_t), &batch_size);
# 闭合前面的函数定义
```