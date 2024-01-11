# `xmrig\src\backend\opencl\kernels\rx\HashAesKernel.h`

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
 * （在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU 通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_HASHAESKERNEL_H
#define XMRIG_HASHAESKERNEL_H


#include "backend/opencl/wrappers/OclKernel.h"


namespace xmrig {


class HashAesKernel : public OclKernel
{
public:
    inline HashAesKernel(cl_program program) : OclKernel(program, "hashAes1Rx4") {}

    void enqueue(cl_command_queue queue, size_t threads);
    void setArgs(cl_mem input, cl_mem hash, uint32_t hashStrideBytes, uint32_t batch_size);
};


} // namespace xmrig


#endif /* XMRIG_HASHAESKERNEL_H */
```