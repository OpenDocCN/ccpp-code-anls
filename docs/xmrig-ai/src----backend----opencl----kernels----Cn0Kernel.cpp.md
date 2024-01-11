# `xmrig\src\backend\opencl\kernels\Cn0Kernel.cpp`

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
 * 由自由软件基金会发布的版本 3 或
 * （根据您的选择）任何以后的版本。
 *
 * 本程序是希望它有用
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU 通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "backend/opencl/kernels/Cn0Kernel.h"
#include "backend/opencl/wrappers/OclLib.h"

// 将任务加入队列
void xmrig::Cn0Kernel::enqueue(cl_command_queue queue, uint32_t nonce, size_t threads)
{
    const size_t offset[2]          = { nonce, 1 };  // 偏移量
    const size_t gthreads[2]        = { threads, 8 };  // 全局线程数
    static const size_t lthreads[2] = { 8, 8 };  // 局部线程数

    // 将任务加入队列
    enqueueNDRange(queue, 2, offset, gthreads, lthreads);
}

// 设置参数
void xmrig::Cn0Kernel::setArgs(cl_mem input, int inlen, cl_mem scratchpads, cl_mem states, uint32_t threads)
{
    setArg(0, sizeof(cl_mem), &input);  // 设置参数0
    setArg(1, sizeof(int), &inlen);  // 设置参数1
    setArg(2, sizeof(cl_mem), &scratchpads);  // 设置参数2
    # 设置第三个参数，表示内存对象的大小，传入states的大小
    setArg(3, sizeof(cl_mem), &states);
    # 设置第四个参数，表示整数的大小，传入threads的大小
    setArg(4, sizeof(uint32_t), &threads);
# 闭合前面的函数定义
```