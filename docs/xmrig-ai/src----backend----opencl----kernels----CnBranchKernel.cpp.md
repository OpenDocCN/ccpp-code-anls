# `xmrig\src\backend\opencl\kernels\CnBranchKernel.cpp`

```
/* XMRig
 * 版权声明，列出了程序的各个作者和版权信息
 */


#include "backend/opencl/kernels/CnBranchKernel.h"
#include "backend/opencl/wrappers/OclLib.h"
// 引入相关头文件


namespace xmrig {
// 命名空间定义


static const char *names[4] = { "Blake", "Groestl", "JH", "Skein" };
// 定义包含四个字符串的静态字符指针数组


} // namespace xmrig
// 命名空间结束


xmrig::CnBranchKernel::CnBranchKernel(size_t index, cl_program program) : OclKernel(program, names[index])
{
}
// CnBranchKernel 类的构造函数，初始化父类 OclKernel 的构造函数


void xmrig::CnBranchKernel::enqueue(cl_command_queue queue, uint32_t nonce, size_t threads, size_t worksize)
{
    const size_t offset   = nonce;
    const size_t gthreads = threads;
    const size_t lthreads = worksize;
    // 定义并初始化三个变量

    enqueueNDRange(queue, 1, &offset, &gthreads, &lthreads);
    // 调用 enqueueNDRange 函数
}


// __kernel void Skein(__global ulong *states, __global uint *BranchBuf, __global uint *output, ulong Target, uint Threads)
// OpenCL 内核函数 Skein 的定义
// 设置内核参数，包括状态、分支、输出和线程数
void xmrig::CnBranchKernel::setArgs(cl_mem states, cl_mem branch, cl_mem output, uint32_t threads)
{
    // 设置第一个参数为状态的内存对象
    setArg(0, sizeof(cl_mem), &states);
    // 设置第二个参数为分支的内存对象
    setArg(1, sizeof(cl_mem), &branch);
    // 设置第三个参数为输出的内存对象
    setArg(2, sizeof(cl_mem), &output);
    // 设置第四个参数为线程数
    setArg(4, sizeof(cl_uint), &threads);
}

// 设置目标值
void xmrig::CnBranchKernel::setTarget(uint64_t target)
{
    // 设置第三个参数为目标值
    setArg(3, sizeof(cl_ulong), &target);
}
```