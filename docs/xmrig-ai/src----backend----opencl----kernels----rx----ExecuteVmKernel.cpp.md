# `xmrig\src\backend\opencl\kernels\rx\ExecuteVmKernel.cpp`

```cpp
// 包含版权声明和许可协议信息
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

// 包含所需的头文件
#include "backend/opencl/kernels/rx/ExecuteVmKernel.h"
#include "backend/opencl/wrappers/OclLib.h"

// 定义函数，将执行虚拟机内核的命令加入到命令队列中
void xmrig::ExecuteVmKernel::enqueue(cl_command_queue queue, size_t threads, size_t worksize)
{
    // 根据工作大小选择全局线程数
    const size_t gthreads = (worksize == 16) ? (threads * 16) : (threads * 8);
    // 根据工作大小选择局部线程数
    const size_t lthreads = (worksize == 16) ? 32 : 16;

    // 将命令加入到命令队列中
    enqueueNDRange(queue, 1, nullptr, &gthreads, &lthreads);
}

// 定义函数，设置执行虚拟机内核所需的参数
void xmrig::ExecuteVmKernel::setArgs(cl_mem vm_states, cl_mem rounding, cl_mem scratchpads, cl_mem dataset_ptr, uint32_t batch_size)
{
    # 设置第一个参数，传入cl_mem类型的数据，数据大小为vm_states的大小，数据地址为vm_states的地址
    setArg(0, sizeof(cl_mem), &vm_states);
    # 设置第二个参数，传入cl_mem类型的数据，数据大小为rounding的大小，数据地址为rounding的地址
    setArg(1, sizeof(cl_mem), &rounding);
    # 设置第三个参数，传入cl_mem类型的数据，数据大小为scratchpads的大小，数据地址为scratchpads的地址
    setArg(2, sizeof(cl_mem), &scratchpads);
    # 设置第四个参数，传入cl_mem类型的数据，数据大小为dataset_ptr的大小，数据地址为dataset_ptr的地址
    setArg(3, sizeof(cl_mem), &dataset_ptr);
    # 设置第五个参数，传入uint32_t类型的数据，数据大小为batch_size的大小，数据地址为batch_size的地址
    setArg(4, sizeof(uint32_t), &batch_size);
# 设置执行虚拟机内核的第一个参数
void xmrig::ExecuteVmKernel::setFirst(uint32_t first)
{
    # 调用setArg函数设置第6个参数的数值和大小
    setArg(6, sizeof(uint32_t), &first);
}

# 设置执行虚拟机内核的迭代次数
void xmrig::ExecuteVmKernel::setIterations(uint32_t num_iterations)
{
    # 调用setArg函数设置第5个参数的数值和大小
    setArg(5, sizeof(uint32_t), &num_iterations);
    # 调用setFirst函数设置第一个参数为1
    setFirst(1);
    # 调用setLast函数设置最后一个参数为0
    setLast(0);
}

# 设置执行虚拟机内核的最后一个参数
void xmrig::ExecuteVmKernel::setLast(uint32_t last)
{
    # 调用setArg函数设置第7个参数的数值和大小
    setArg(7, sizeof(uint32_t), &last);
}
```