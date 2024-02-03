# `xmrig\src\backend\opencl\kernels\rx\RxRunKernel.cpp`

```cpp
/*
 * XMRig
 * 版权所有 2010      Jeff Garzik <jgarzik@pobox.com>
 * 版权所有 2012-2014 pooler      <pooler@litecoinpool.org>
 * 版权所有 2014      Lucas Jones <https://github.com/lucasjones>
 * 版权所有 2014-2016 Wolf9466    <https://github.com/OhGodAPet>
 * 版权所有 2016      Jay D Dee   <jayddee246@gmail.com>
 * 版权所有 2017-2018 XMR-Stak    <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * 版权所有 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改
 * 根据 GNU 通用公共许可证的条款发布，由
 * 自由软件基金会发布的版本 3 或
 * （根据您的选择）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证。
 * 适销性或特定用途的保证。请参阅
 * GNU 通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "backend/opencl/kernels/rx/RxRunKernel.h"
#include "backend/opencl/wrappers/OclLib.h"
#include "base/crypto/Algorithm.h"
#include "crypto/randomx/randomx.h"
#include "crypto/rx/RxAlgo.h"

// 将任务排入队列
void xmrig::RxRunKernel::enqueue(cl_command_queue queue, size_t threads, size_t workgroup_size)
{
    const size_t gthreads        = threads * workgroup_size;
    // 将任务排入队列
    enqueueNDRange(queue, 1, nullptr, &gthreads, &workgroup_size);
}

// 设置参数
void xmrig::RxRunKernel::setArgs(cl_mem dataset, cl_mem scratchpads, cl_mem registers, cl_mem rounding, cl_mem programs, uint32_t batch_size, const Algorithm &algorithm)
{
    // 设置参数
    setArg(0, sizeof(cl_mem), &dataset);
    setArg(1, sizeof(cl_mem), &scratchpads);
    setArg(2, sizeof(cl_mem), &registers);
    # 设置第3个参数，传入cl_mem类型的数据，内容为rounding的大小
    setArg(3, sizeof(cl_mem), &rounding);
    # 设置第4个参数，传入cl_mem类型的数据，内容为programs的大小
    setArg(4, sizeof(cl_mem), &programs);
    # 设置第5个参数，传入uint32_t类型的数据，内容为batch_size的大小
    setArg(5, sizeof(uint32_t), &batch_size);

    # 定义一个lambda函数PowerOf2，用于计算一个数的2的幂次方
    auto PowerOf2 = [](size_t N)
    {
        uint32_t result = 0;
        while (N > 1) {
            ++result;
            N >>= 1;
        }

        return result;
    };

    # 获取指定算法的配置信息
    const auto *rx_conf = RxAlgo::base(algorithm);
    # 计算rx_parameters，包括ScratchpadL1_Size、ScratchpadL2_Size、ScratchpadL3_Size和ProgramIterations的2的幂次方
    const uint32_t rx_parameters =
                    (PowerOf2(rx_conf->ScratchpadL1_Size) << 0) |
                    (PowerOf2(rx_conf->ScratchpadL2_Size) << 5) |
                    (PowerOf2(rx_conf->ScratchpadL3_Size) << 10) |
                    (PowerOf2(rx_conf->ProgramIterations) << 15);

    # 设置第6个参数，传入uint32_t类型的数据，内容为rx_parameters的大小
    setArg(6, sizeof(uint32_t), &rx_parameters);
# 闭合前面的函数定义
```