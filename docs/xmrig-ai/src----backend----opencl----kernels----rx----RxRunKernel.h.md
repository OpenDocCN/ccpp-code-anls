# `xmrig\src\backend\opencl\kernels\rx\RxRunKernel.h`

```
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
 *   本程序是自由软件：您可以重新发布和/或修改
 *   根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 *   要么许可证的第3版，要么（按您的选择）任何以后的版本。
 *
 *   本程序是希望它有用：
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。更多细节请参见
 *   GNU 通用公共许可证。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_RXRUNKERNEL_H
#define XMRIG_RXRUNKERNEL_H


#include "backend/opencl/wrappers/OclKernel.h"


namespace xmrig {


class Algorithm;


class RxRunKernel : public OclKernel
{
public:
    inline RxRunKernel(cl_program program) : OclKernel(program, "randomx_run") {}

    void enqueue(cl_command_queue queue, size_t threads, size_t workgroup_size);
    void setArgs(cl_mem dataset, cl_mem scratchpads, cl_mem registers, cl_mem rounding, cl_mem programs, uint32_t batch_size, const Algorithm &algorithm);
};


} // namespace xmrig


#endif /* XMRIG_RXRUNKERNEL_H */
```