# `xmrig\src\backend\opencl\kernels\Cn1Kernel.h`

```
// 包含版权声明和许可信息
/* XMRig
 * Copyright 2010      Jeff Garzik <jgarzik@pobox.com>
 * Copyright 2012-2014 pooler      <pooler@litecoinpool.org>
 * Copyright 2014      Lucas Jones <https://github.com/lucasjones>
 * Copyright 2014-2016 Wolf9466    <https://github.com/OhGodAPet>
 * Copyright 2016      Jay D Dee   <jayddee246@gmail.com>
 * Copyright 2017-2018 XMR-Stak    <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * Copyright 2018-2019 SChernykh   <https://github.com/SChernykh>
 * Copyright 2016-2019 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   This program is free software: you can redistribute it and/or modify
 *   it under the terms of the GNU General Public License as published by
 *   the Free Software Foundation, either version 3 of the License, or
 *   (at your option) any later version.
 *
 *   This program is distributed in the hope that it will be useful,
 *   but WITHOUT ANY WARRANTY; without even the implied warranty of
 *   MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
 *   GNU General Public License for more details.
 *
 *   You should have received a copy of the GNU General Public License
 *   along with this program. If not, see <http://www.gnu.org/licenses/>.
 */

// 防止头文件重复包含
#ifndef XMRIG_CN1KERNEL_H
#define XMRIG_CN1KERNEL_H

// 包含 OpenCL 内核类的头文件
#include "backend/opencl/wrappers/OclKernel.h"

// 命名空间声明
namespace xmrig {

// 定义 Cn1Kernel 类，继承自 OclKernel 类
class Cn1Kernel : public OclKernel
{
public:
    // 构造函数，接受 OpenCL 程序作为参数
    Cn1Kernel(cl_program program);
    // 构造函数，接受 OpenCL 程序和块高度作为参数
    Cn1Kernel(cl_program program, uint64_t height);

    // 将任务加入到命令队列中
    void enqueue(cl_command_queue queue, uint32_t nonce, size_t threads, size_t worksize);
    // 设置内核参数
    void setArgs(cl_mem input, cl_mem scratchpads, cl_mem states, uint32_t threads);
};

} // 命名空间结束

// 结束条件编译指令
#endif /* XMRIG_CN1KERNEL_H */
```