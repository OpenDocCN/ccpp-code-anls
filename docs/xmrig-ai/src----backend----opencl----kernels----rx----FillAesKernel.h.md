# `xmrig\src\backend\opencl\kernels\rx\FillAesKernel.h`

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
 * 本程序是自由软件：您可以重新发布或修改
 * 根据 GNU 通用公共许可证的条款发布
 * 由自由软件基金会发布的许可证的第3版或
 * （根据您的选择）任何以后的版本。
 *
 * 本程序是希望它有用
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU 通用公共许可证以获取更多详细信息。
 *
 * 您应该已收到 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_FILLAESKERNEL_H
#define XMRIG_FILLAESKERNEL_H


#include "backend/opencl/wrappers/OclKernel.h"


namespace xmrig {


class FillAesKernel : public OclKernel
{
public:
    inline FillAesKernel(cl_program program, const char *name) : OclKernel(program, name) {}

    void enqueue(cl_command_queue queue, size_t threads);
    void setArgs(cl_mem state, cl_mem out, uint32_t batch_size, uint32_t rx_version);
};


} // namespace xmrig


#endif /* XMRIG_FILLAESKERNEL_H */
```