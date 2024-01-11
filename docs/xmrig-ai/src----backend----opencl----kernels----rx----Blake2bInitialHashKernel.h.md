# `xmrig\src\backend\opencl\kernels\rx\Blake2bInitialHashKernel.h`

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
 * 根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 * 许可证的版本为 3 或（按您的选择）更高版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证适用于特定用途。详细信息请参见
 * GNU 通用公共许可证。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_BLAKE2BINITIALHASHKERNEL_H
#define XMRIG_BLAKE2BINITIALHASHKERNEL_H

#include "backend/opencl/wrappers/OclKernel.h"

namespace xmrig {

class Blake2bInitialHashKernel : public OclKernel
{
public:
    inline Blake2bInitialHashKernel(cl_program program) : OclKernel(program, "blake2b_initial_hash") {} // 使用给定的程序和内核名称初始化对象

    void enqueue(cl_command_queue queue, size_t threads); // 将内核排入命令队列以供执行
    void setArgs(cl_mem out, cl_mem blockTemplate); // 设置内核参数
    void setBlobSize(size_t size); // 设置内核参数
    void setNonce(uint32_t nonce); // 设置内核参数
};

} // namespace xmrig

#endif /* XMRIG_BLAKE2BINITIALHASHKERNEL_H */
```