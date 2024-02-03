# `xmrig\src\backend\opencl\kernels\rx\Blake2bHashRegistersKernel.h`

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
 * 本程序是自由软件：您可以根据 GNU 通用公共许可证的条款重新分发或修改它，无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是基于希望它有用的前提下分发的，但没有任何担保；甚至没有适销性或特定用途的暗示担保。有关更多详细信息，请参见 GNU 通用公共许可证。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_BLAKE2BHASHREGISTERSKERNEL_H
#define XMRIG_BLAKE2BHASHREGISTERSKERNEL_H

#include "backend/opencl/wrappers/OclKernel.h"

namespace xmrig {

class Blake2bHashRegistersKernel : public OclKernel
{
public:
    // 构造函数，初始化父类 OclKernel
    inline Blake2bHashRegistersKernel(cl_program program, const char *name) : OclKernel(program, name) {}

    // 将任务加入到命令队列中
    void enqueue(cl_command_queue queue, size_t threads);
    // 设置内核参数
    void setArgs(cl_mem out, cl_mem in, uint32_t inStrideBytes);
};

} // namespace xmrig

#endif /* XMRIG_BLAKE2BHASHREGISTERSKERNEL_H */
```