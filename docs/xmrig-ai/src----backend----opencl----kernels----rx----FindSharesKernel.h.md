# `xmrig\src\backend\opencl\kernels\rx\FindSharesKernel.h`

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
 *   本程序是自由软件：您可以重新发布或修改
 *   根据 GNU 通用公共许可证的条款发布，由
 *   自由软件基金会发布的许可证的第3版或
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用：
 *   但没有任何保证；甚至没有暗示的保证
 *   商品性或适用于特定目的。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_FINDSHARESKERNEL_H
#define XMRIG_FINDSHARESKERNEL_H


#include "backend/opencl/wrappers/OclKernel.h"


namespace xmrig {


class FindSharesKernel : public OclKernel
{
public:
    inline FindSharesKernel(cl_program program) : OclKernel(program, "find_shares") {}

    void enqueue(cl_command_queue queue, size_t threads);
    void setArgs(cl_mem hashes, cl_mem shares);
    void setTarget(uint64_t target);
    void setNonce(uint32_t nonce);
};


} // namespace xmrig


#endif /* XMRIG_FINDSHARESKERNEL_H */
```