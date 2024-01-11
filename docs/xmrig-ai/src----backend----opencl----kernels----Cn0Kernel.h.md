# `xmrig\src\backend\opencl\kernels\Cn0Kernel.h`

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
 * 根据 GNU 通用公共许可证的条款发布，
 * 由自由软件基金会发布的版本 3 或
 * （根据您的选择）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。详细信息请参见
 * GNU 通用公共许可证。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_CN0KERNEL_H
#define XMRIG_CN0KERNEL_H


#include "backend/opencl/wrappers/OclKernel.h"


namespace xmrig {


class Cn0Kernel : public OclKernel
{
public:
    inline Cn0Kernel(cl_program program) : OclKernel(program, "cn0") {}  // 使用给定的程序和内核名称初始化Cn0Kernel对象

    void enqueue(cl_command_queue queue, uint32_t nonce, size_t threads);  // 将命令添加到队列以执行内核
    void setArgs(cl_mem input, int inlen, cl_mem scratchpads, cl_mem states, uint32_t threads);  // 设置内核参数
};


} // namespace xmrig


#endif /* XMRIG_CN0KERNEL_H */
```