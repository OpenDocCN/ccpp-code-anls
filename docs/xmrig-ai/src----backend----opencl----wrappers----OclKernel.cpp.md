# `xmrig\src\backend\opencl\wrappers\OclKernel.cpp`

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

#include "backend/common/Tags.h"  // 引入 Tags.h 文件
#include "backend/opencl/wrappers/OclError.h"  // 引入 OclError.h 文件
#include "backend/opencl/wrappers/OclKernel.h"  // 引入 OclKernel.h 文件
#include "backend/opencl/wrappers/OclLib.h"  // 引入 OclLib.h 文件
#include "base/io/log/Log.h"  // 引入 Log.h 文件

#include <stdexcept>  // 引入标准异常库

// 使用 xmrig 命名空间，定义 OclKernel 类的构造函数
xmrig::OclKernel::OclKernel(cl_program program, const char *name) :
    m_name(name)  // 初始化成员变量 m_name
{
    m_kernel = OclLib::createKernel(program, name);  // 调用 OclLib 类的 createKernel 方法创建内核
}

// 定义 OclKernel 类的析构函数
xmrig::OclKernel::~OclKernel()
{
    OclLib::release(m_kernel);  // 调用 OclLib 类的 release 方法释放内核
}

// 定义 OclKernel 类的 enqueueNDRange 方法
void xmrig::OclKernel::enqueueNDRange(cl_command_queue queue, uint32_t work_dim, const size_t *global_work_offset, const size_t *global_work_size, const size_t *local_work_size)
{
    // 调用 OpenCL 库中的 enqueueNDRangeKernel 函数执行内核
    const cl_int ret = OclLib::enqueueNDRangeKernel(queue, m_kernel, work_dim, global_work_offset, global_work_size, local_work_size, 0, nullptr, nullptr);
    // 如果返回值不是 CL_SUCCESS，则表示执行内核出错
    if (ret != CL_SUCCESS) {
        // 记录错误日志
        LOG_ERR("%s" RED(" error ") RED_BOLD("%s") RED(" when calling ") RED_BOLD("clEnqueueNDRangeKernel") RED(" for kernel ") RED_BOLD("%s"),
                ocl_tag(), OclError::toString(ret), name().data());
        // 抛出运行时异常，包含错误信息
        throw std::runtime_error(OclError::toString(ret));
    }
// 设置 OpenCL 内核函数的参数
void xmrig::OclKernel::setArg(uint32_t index, size_t size, const void *value)
{
    // 调用 OclLib::setKernelArg 函数设置内核函数的参数，并返回结果
    const cl_int ret = OclLib::setKernelArg(m_kernel, index, size, value);
    // 如果返回结果不是 CL_SUCCESS，则记录错误信息并抛出异常
    if (ret != CL_SUCCESS) {
        LOG_ERR("%s" RED(" error ") RED_BOLD("%s") RED(" when calling ") RED_BOLD("clSetKernelArg") RED(" for kernel ") RED_BOLD("%s")
                RED(" argument ") RED_BOLD("%u") RED(" size ") RED_BOLD("%zu"),
                ocl_tag(), OclError::toString(ret), name().data(), index, size);

        throw std::runtime_error(OclError::toString(ret));
    }
}
```