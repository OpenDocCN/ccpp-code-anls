# `xmrig\src\backend\opencl\kernels\kawpow\KawPow_CalculateDAGKernel.cpp`

```
/*
 * XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新分发或修改
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款
 * 许可证的第3版或（根据您的选择）任何以后的版本。
 *
 * 本程序是希望它有用的分发，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "KawPow_CalculateDAGKernel.h"
#include "backend/opencl/wrappers/OclLib.h"
#include "crypto/kawpow/KPCache.h"


void xmrig::KawPow_CalculateDAGKernel::enqueue(cl_command_queue queue, size_t threads, size_t workgroup_size)
{
    // 将计算任务加入到命令队列中，使用指定的线程数和工作组大小
    enqueueNDRange(queue, 1, nullptr, &threads, &workgroup_size);
}


void xmrig::KawPow_CalculateDAGKernel::setArgs(uint32_t start, cl_mem g_light, cl_mem g_dag, uint32_t dag_words, uint32_t light_words)
{
    // 设置内核参数
    setArg(0, sizeof(start), &start);
    setArg(1, sizeof(cl_mem), &g_light);
    setArg(2, sizeof(cl_mem), &g_dag);

    const uint32_t isolate = 1;
    setArg(3, sizeof(isolate), &isolate);

    setArg(4, sizeof(dag_words), &dag_words);

    uint32_t light_words4[4];
    // 计算快速模数数据
    KPCache::calculate_fast_mod_data(light_words, light_words4[0], light_words4[1], light_words4[2]);
    light_words4[3] = light_words;

    setArg(5, sizeof(light_words4), light_words4);
}
```