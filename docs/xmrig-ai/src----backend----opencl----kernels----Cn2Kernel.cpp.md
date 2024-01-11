# `xmrig\src\backend\opencl\kernels\Cn2Kernel.cpp`

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
 * 根据 GNU 通用公共许可证的条款发布，由
 * 自由软件基金会发布的版本 3 或
 * （根据您的选择）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU 通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#include "backend/opencl/kernels/Cn2Kernel.h"
#include "backend/opencl/wrappers/OclLib.h"

// 将 Cn2Kernel 类的 enqueue 方法定义
void xmrig::Cn2Kernel::enqueue(cl_command_queue queue, uint32_t nonce, size_t threads)
{
    // 定义 offset 数组，包含 nonce 和 1
    const size_t offset[2]          = { nonce, 1 };
    // 定义 gthreads 数组，包含 threads 和 8
    const size_t gthreads[2]        = { threads, 8 };
    // 定义静态的 lthreads 数组，包含 8 和 8
    static const size_t lthreads[2] = { 8, 8 };

    // 调用 enqueueNDRange 方法，传入队列、维度、offset、gthreads 和 lthreads
    enqueueNDRange(queue, 2, offset, gthreads, lthreads);
}

// 将 Cn2Kernel 类的 setArgs 方法定义
void xmrig::Cn2Kernel::setArgs(cl_mem scratchpads, cl_mem states, const std::vector<cl_mem> &branches, uint32_t threads)
{
    // 设置第一个参数为 cl_mem 类型，值为 scratchpads
    setArg(0, sizeof(cl_mem), &scratchpads);
    # 设置第一个参数，表示内存对象的大小，传入状态
    setArg(1, sizeof(cl_mem), &states);
    # 设置第六个参数，表示无符号整数的大小，传入线程数
    setArg(6, sizeof(uint32_t), &threads);

    # 遍历分支列表，设置参数，参数索引从2开始
    for (uint32_t i = 0; i < branches.size(); ++i) {
        setArg(i + 2, sizeof(cl_mem), &branches[i]);
    }
# 闭合前面的函数定义
```