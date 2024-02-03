# `xmrig\src\backend\opencl\runners\OclBaseRunner.cpp`

```cpp
/* XMRig
 * 版权所有 2010      Jeff Garzik <jgarzik@pobox.com>
 * 版权所有 2012-2014 pooler      <pooler@litecoinpool.org>
 * 版权所有 2014      Lucas Jones <https://github.com/lucasjones>
 * 版权所有 2014-2016 Wolf9466    <https://github.com/OhGodAPet>
 * 版权所有 2016      Jay D Dee   <jayddee246@gmail.com>
 * 版权所有 2017-2018 XMR-Stak    <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * 版权所有 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据 GNU 通用公共许可证的条款发布，由
 *   自由软件基金会发布的版本 3 或
 *   （根据您的选择）任何以后的版本。
 *
 *   本程序是希望它有用：
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。更多细节请参见
 *   GNU 通用公共许可证。
 *
 *   您应该已经收到 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */


#include <stdexcept>


#include "backend/opencl/runners/OclBaseRunner.h"
#include "backend/opencl/cl/OclSource.h"
#include "backend/opencl/OclCache.h"
#include "backend/opencl/OclLaunchData.h"
#include "backend/opencl/runners/tools/OclSharedState.h"
#include "backend/opencl/wrappers/OclError.h"
#include "backend/opencl/wrappers/OclLib.h"
#include "base/io/log/Log.h"
#include "base/net/stratum/Job.h"
#include "crypto/common/VirtualMemory.h"


// 定义一个常量，表示1GB的大小
constexpr size_t oneGiB = 1024 * 1024 * 1024;


// 构造函数，初始化成员变量
xmrig::OclBaseRunner::OclBaseRunner(size_t id, const OclLaunchData &data) :
    m_ctx(data.ctx),  // 初始化 m_ctx 成员变量
    m_algorithm(data.algorithm),  // 初始化 m_algorithm 成员变量
    m_source(OclSource::get(data.algorithm)),  // 使用 OclSource 类的 get 方法初始化 m_source 成员变量
    m_data(data),  // 初始化 m_data 成员变量
    # 调用 OclLib::getUint 方法获取设备内存基地址对齐值，并传递给 m_align
    m_align(OclLib::getUint(data.device.id(), CL_DEVICE_MEM_BASE_ADDR_ALIGN)),
    # 将 id 传递给 m_threadId
    m_threadId(id),
    # 调用 data.thread.intensity() 方法获取线程强度值，并传递给 m_intensity
    m_intensity(data.thread.intensity())
{
    // 获取设备名称并赋值给 m_deviceKey
    m_deviceKey = data.device.name();

#   ifdef XMRIG_STRICT_OPENCL_CACHE
    // 如果定义了 XMRIG_STRICT_OPENCL_CACHE，则将平台版本追加到 m_deviceKey
    m_deviceKey += ":";
    m_deviceKey += data.platform.version();

    // 将设备驱动版本追加到 m_deviceKey
    m_deviceKey += ":";
    m_deviceKey += OclLib::getString(data.device.id(), CL_DRIVER_VERSION);
#   endif

#   if defined(__x86_64__) || defined(_M_AMD64) || defined (__arm64__) || defined (__aarch64__)
    // 如果是64位架构，则在 m_deviceKey 后面追加 ":64"
    m_deviceKey += ":64";
#   endif
}


// OclBaseRunner 类的析构函数
xmrig::OclBaseRunner::~OclBaseRunner()
{
    // 释放程序对象
    OclLib::release(m_program);
    // 释放输入对象
    OclLib::release(m_input);
    // 释放输出对象
    OclLib::release(m_output);
    // 释放缓冲区对象
    OclLib::release(m_buffer);
    // 释放命令队列对象
    OclLib::release(m_queue);
}


// 返回缓冲区大小
size_t xmrig::OclBaseRunner::bufferSize() const
{
    return align(Job::kMaxBlobSize) + align(sizeof(cl_uint) * 0x100);
}


// 返回设备索引
uint32_t xmrig::OclBaseRunner::deviceIndex() const
{
    return data().thread.index();
}


// 构建程序对象
void xmrig::OclBaseRunner::build()
{
    m_program = OclCache::build(this);

    // 如果程序对象为空，则抛出运行时错误
    if (m_program == nullptr) {
        throw std::runtime_error(OclError::toString(CL_INVALID_PROGRAM));
    }
}


// 初始化
void xmrig::OclBaseRunner::init()
{
    // 创建命令队列对象
    m_queue = OclLib::createCommandQueue(m_ctx, data().device.id());

    size_t size         = align(bufferSize());
    const size_t limit  = data().device.freeMemSize();

    // 如果满足条件，则创建缓冲区对象
    if (size < oneGiB && data().device.vendorId() == OCL_VENDOR_AMD && limit >= oneGiB) {
        m_buffer = OclSharedState::get(data().device.index()).createBuffer(m_ctx, size, m_offset, limit);
    }

    // 如果缓冲区对象为空，则创建缓冲区对象
    if (!m_buffer) {
        m_buffer = OclLib::createBuffer(m_ctx, CL_MEM_READ_WRITE, size);
    }

    // 创建输入子缓冲区对象
    m_input  = createSubBuffer(CL_MEM_READ_ONLY | CL_MEM_HOST_WRITE_ONLY, Job::kMaxBlobSize);
    // 创建输出子缓冲区对象
    m_output = createSubBuffer(CL_MEM_READ_WRITE, sizeof(cl_uint) * 0x100);
}


// 创建子缓冲区对象
cl_mem xmrig::OclBaseRunner::createSubBuffer(cl_mem_flags flags, size_t size)
{
    auto mem = OclLib::createSubBuffer(m_buffer, flags, m_offset, size);

    m_offset += align(size);

    return mem;
}


// 对齐大小
size_t xmrig::OclBaseRunner::align(size_t size) const
{
    return VirtualMemory::align(size, m_align);
}
// 将数据从 OpenCL 缓冲区读取到主机内存中
void xmrig::OclBaseRunner::enqueueReadBuffer(cl_mem buffer, cl_bool blocking_read, size_t offset, size_t size, void *ptr)
{
    // 调用 OpenCL 库函数将数据从缓冲区读取到主机内存中
    const cl_int ret = OclLib::enqueueReadBuffer(m_queue, buffer, blocking_read, offset, size, ptr, 0, nullptr, nullptr);
    // 如果返回值不是 CL_SUCCESS，则抛出运行时错误
    if (ret != CL_SUCCESS) {
        throw std::runtime_error(OclError::toString(ret));
    }
}

// 将数据从主机内存写入到 OpenCL 缓冲区中
void xmrig::OclBaseRunner::enqueueWriteBuffer(cl_mem buffer, cl_bool blocking_write, size_t offset, size_t size, const void *ptr)
{
    // 调用 OpenCL 库函数将数据从主机内存写入到缓冲区中
    const cl_int ret = OclLib::enqueueWriteBuffer(m_queue, buffer, blocking_write, offset, size, ptr, 0, nullptr, nullptr);
    // 如果返回值不是 CL_SUCCESS，则抛出运行时错误
    if (ret != CL_SUCCESS) {
        throw std::runtime_error(OclError::toString(ret));
    }
}

// 完成 OpenCL 运行，将结果从输出缓冲区读取到主机内存中
void xmrig::OclBaseRunner::finalize(uint32_t *hashOutput)
{
    // 将输出缓冲区的数据读取到主机内存中
    enqueueReadBuffer(m_output, CL_TRUE, 0, sizeof(cl_uint) * 0x100, hashOutput);

    // 获取结果中的最后一个元素
    uint32_t &results = hashOutput[0xFF];
    // 如果结果大于 0xFF，则将其设置为 0xFF
    if (results > 0xFF) {
        results = 0xFF;
    }
}
```