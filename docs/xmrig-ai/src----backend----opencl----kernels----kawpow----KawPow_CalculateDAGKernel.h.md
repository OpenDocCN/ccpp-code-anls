# `xmrig\src\backend\opencl\kernels\kawpow\KawPow_CalculateDAGKernel.h`

```cpp
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款
 * 它可以是许可证的第3版，也可以是（根据您的选择）任何以后的版本。
 *
 * 本程序是希望它有用
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。有关更多详细信息，请参见
 * GNU通用公共许可证。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_KAWPOW_CALCULATEDAGKERNEL_H
#define XMRIG_KAWPOW_CALCULATEDAGKERNEL_H


#include "backend/opencl/wrappers/OclKernel.h"


namespace xmrig {


class KawPow_CalculateDAGKernel : public OclKernel
{
public:
    inline KawPow_CalculateDAGKernel(cl_program program) : OclKernel(program, "ethash_calculate_dag_item") {}

    void enqueue(cl_command_queue queue, size_t threads, size_t workgroup_size);
    void setArgs(uint32_t start, cl_mem g_light, cl_mem g_dag, uint32_t dag_words, uint32_t light_words);
};


} // namespace xmrig


#endif /* XMRIG_KAWPOW_CALCULATEDAGKERNEL_H */
```